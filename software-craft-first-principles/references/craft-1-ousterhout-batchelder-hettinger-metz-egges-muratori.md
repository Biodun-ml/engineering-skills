# craft-1 — cross-cutting principles

1. **Design is decomposition to reduce complexity.** Either eliminate complexity outright (design away special cases) or hide it in *deep modules*: a small interface over a lot of functionality. Judge every module by how much it hides compared with how much callers must learn. (Ousterhout)
2. **Depth beats shortness or "do one thing".** Splitting code into many tiny, entangled methods or services multiplies interfaces. Combining closely related pieces often gives a deeper, simpler unit. Any rule pushed to an extreme goes bad; design is trade-offs. (Ousterhout; Casey Muratori makes the performance version of the same argument)
3. **Design it twice.** Before building, sketch a genuinely different second approach and compare. It costs about 1–2% of project time and often wins. (Ousterhout)
4. **Wrap ugly external APIs in an adapter.** The adapter changes the API's shape, not the work: a context manager for setup/teardown and commit/rollback, properties instead of getters, `__len__`/`__getitem__`/`__iter__` instead of size/index methods, named custom exceptions replacing unhelpful ones, and a `__repr__`. Business logic then reads clearly. (Hettinger; the same idea as Ousterhout's deep module)
5. **Design errors from the caller's side.** Every exception is complexity exported to callers. Collapse many error types into the few reactions callers actually have, and design away errors where possible. But never ignore real failures (network, crashes), and never map an unknown response to success. (Ousterhout; Hettinger on translating exceptions)
6. **Duplication is far cheaper than the wrong abstraction.** Tolerate duplication (tag it, e.g. `# DUP-3`) until real cases reveal the right abstraction. Finish the current refactor before chasing a DRY tangent. (Metz)
7. **Make the change easy, then make the easy change.** Let a real new requirement drive refactoring. Aim for open/closed code where new behaviour means *adding* a class or registry entry, not editing a conditional. (Metz, quoting Kent Beck)
8. **Refactor safely: tests at your back, then seam, then green, then refactor, then delete.** Wrap legacy code in characterization tests, route one path at a time to new code with dumb code that passes, clean up on green, and delete the old code without needing to understand it. Expect intermediate steps to look *worse*. (Metz)
9. **Type-switch to factory plus polymorphism, and mappings to config.** A conditional over a type code becomes a factory that picks the class plus one small class per variant. The name→class mapping is configuration (a dict, then YAML or a DB), separate from the algorithm. Inheritance only when shallow, narrow, at leaf nodes, with subclasses using all the parent's behaviour. (Metz)
10. **...but organise hot loops by operation and data, not by type hierarchy.** Per-element virtual dispatch through pointers was 1.5x slower than a switch and 10–15x slower than a lookup table in Casey's benchmark (20x+ against simple SIMD). Put polymorphism at the boundaries (one call per supplier or batch) and keep inner loops data-oriented. In Python, that means vectorised column operations and lookup arrays instead of per-row objects and `.apply`. (Casey Muratori; Python translation is my synthesis)
11. **Inject dependencies; wire in `main`.** Components receive their loader, client and exporter (typed as `Protocol`s or `Callable`s that specify methods, never `__init__`), and a composition root builds them. Use functions for single-method dependencies. Use DI containers only for config-driven wiring; FastAPI `Depends` for endpoints; plain function calls for small scripts. (Arjan; Ned on DI vs monkeypatching)
12. **Tests solve a real problem; treat them as real code.** Good tests are automated, fast, reliable, informative and focused (each runs as little code as possible). Each test is isolated and order-independent. Refactor tests with domain-specific assert helpers and base classes; DRY applies. (Ned Batchelder)
13. **Mocks are useful but fragile.** They couple tests to implementation details (which module or function is called) and can drift from the real external format, so tests pass while production fails. Prefer injected fakes, assert on call arguments, and refresh recorded fixtures. (Ned; Arjan)
14. **Test-first where it pays.** Ousterhout wants high-coverage unit tests but rejects TDD as a *design* method: it makes one test the unit of design, producing tactical point solutions. Make abstractions the unit of work. Writing the failing test first *for bug fixes* is good (or fix, test, revert the fix to confirm it fails). Metz's use of TDD in refactoring (red means scramble to green) is compatible: it works inside an already-chosen design. (Ousterhout; Metz)
15. **Comments say what code can't.** Interface comments first (callers should never need to read the implementation), then member/field docs, then tricky spots and lessons from bugs. Delete comments that repeat the code. AI can partly explain uncommented code, but that's no excuse to skip comments. (Ousterhout; Hettinger on useless trailing comments)
16. **Names carry more weight in Python** (no type declarations next to variables). Parameter names are public API (keyword arguments). Use keyword args at unclear call sites and namedtuples or dataclasses instead of positional tuples. (Hettinger)
17. **PEP 8 is a guide, not a weapon.** Automate formatting, use roughly 90-character lines, and spend review attention on the "gorilla": non-Pythonic API shape, naming and error semantics. Don't reformat others' code (blame churn, new bugs). (Hettinger; mCoding)
18. **Basic Python hygiene that prevents real bugs:** context managers for resources, no mutable defaults, no bare `except`, no `shell=True`, `isinstance` not `type ==`, `logging` not print, `perf_counter` for timing, install as a package instead of relying on `sys.path` luck. (mCoding)
19. **Thin, testable entry points:** `if __name__ == "__main__": sys.exit(main(sys.argv))`, where `main(argv) -> int` is a plain function. All I/O wiring sits at the edge. (Ned; Arjan's composition root) (note: with argparse, write `def main(argv: Sequence[str] | None = None) -> int` calling `parser.parse_args(argv)` and `raise SystemExit(main())`; passing all of `sys.argv` makes argparse parse the program name.)
20. **In the AI era, design is the scarce skill.** AI tools write decent tactical code but not the decomposition. The human's job is to choose interfaces, compare alternatives and review for shallow modules, premature abstractions, swallowed errors and row-wise slowness. Beware the "tactical tornado", human or AI. (Ousterhout, applied)

---

## John Ousterhout (with Gergely Orosz) — The Philosophy of Software Design (https://www.youtube.com/watch?v=lz451zUlF-k)
**Design is decomposition aimed at reducing complexity: either eliminate it, or hide it behind deep modules with simple interfaces.**

- Mental models / first principles
  - Software design = how you split a large system into units that can be built fairly independently. Ousterhout calls decomposition the single most important idea in computer science.
  - Two ways to fight complexity: (1) eliminate it outright (e.g. design away special cases), the most powerful option; (2) hide what's left in a module so the rest of the system never has to know about it.
  - Deep vs shallow modules: depth = functionality delivered divided by interface complexity. A deep module offers a tiny, easy-to-learn interface over a lot of hidden behaviour. A shallow module has a wide interface and does little, so it adds cognitive load without hiding anything.
  - "Tactical tornado": the prolific coder who ships features fast and leaves a mess others clean up. Managers often mistake this for a 10x engineer. His real 10x engineer produces clean designs that need *less* code, with more stability and evolvability.
  - Strategic vs tactical programming: design is about the long term; the tactical style optimises the next feature only.
  - On AI: tools will increasingly produce decent low-level code, so the share of a developer's time spent on design grows. He sees no sign (as of early 2025) that tools can do the high-level decomposition. So design skill becomes *more* valuable, not less.

- Mechanisms
  - Design it twice: before committing, force yourself to come up with a radically different second approach, even one you think is bad, and compare. His Tk widget API (a big reason Tcl/Tk took off) was his *second* idea, sketched on a plane. Cost: a few days of high-level thinking against a year of implementation, roughly 1–2% of project time, and it pays back far more.
  - His trick for students: "suppose you're forbidden from doing what you just proposed; what's plan B?" The second idea is nearly always better. Smart people who always got by on their first idea have to unlearn that habit.
  - Error handling: every exception you throw is complexity forced onto callers. Sometimes a small design change makes a whole class of errors impossible ("define errors out of existence"). Think from the caller's side: if callers will only ever react in 2 ways, collapse 10 exception types into 2.
  - Comments: say what the code cannot say. The most important place is interface comments (what a caller needs, so they never read the implementation), then documentation of class member variables. Inside method bodies, comment only tricky or surprising things, like lessons from bugs. Delete comments that just repeat the code.
  - Bug-fix testing: write a test that catches the bug. His variant: write the fix, write the test, back the fix out to check the test fails, then put the fix back.

- Decisions & trade-offs
  - Top-down vs bottom-up: neither works pure. Real design goes back and forth: rough big pieces, build some small pieces, find what's wrong, throw it away, revise.
  - Upfront design vs just coding: always do *some* design up front as a working hypothesis (not waterfall), and expect to revise it once implementation exposes problems. Design runs through coding, testing and bug fixing too. Coding with no design only makes sense for a novice learning by doing.
  - Short methods (vs Clean Code): length is not the goal; depth is. Splitting into many 3-line methods multiplies interfaces, and if the pieces are entangled (you can't understand one without reading the others) you've made things worse. Combining closely related things often gives a *deeper* unit with a simpler overall API. Same lesson as Uber's 5,000 microservices drifting back to mid-size domain services.
  - Single-responsibility and "do one thing" push in one direction with no limit, which leads to over-decomposition. Depth captures the trade-off in both directions.
  - TDD: he's against it as a *design* method, not against tests. He wants high-coverage unit tests on everything. TDD makes the unit of development one test, so you build point solutions and never step back for the general design. The unit of development should be an *abstraction*. The only benefit he grants TDD is that tests actually get written.
  - Generality: push towards general-purpose designs and away from special cases. This was his biggest addition in the 2nd edition, learned from years of watching student projects.
  - Familiar domain vs new domain: predictability depends on whether you've crossed this terrain before. New domains and immature frameworks are never smooth, so plan in slack and expect to redesign.

- Rules of thumb / checklist
  - For each module ask: how much does it hide compared with how much a caller must learn?
  - Write down at least one alternative design (an "alternatives considered" section) before building anything non-trivial.
  - Design exceptions for the caller: how many distinct reactions are really needed?
  - Write interface comments first and well. If an interface is hard to describe, the design is probably wrong.
  - Switch viewpoint on purpose: implementer (all the details) vs user (only the interface; never rely on internals you happen to know).
  - Design reviews: several minds beat one. Avoid analysis paralysis.
  - Whiteboard technique for deadlocked debates: list every argument for and against; nobody may strike an argument, nobody may repeat one already listed; when ideas run out, take a straw poll with each person weighting the arguments. Among goal-aligned teams this usually produces strong consensus.
  - Learning loop: build, get detailed critique, redo. The redo is where lessons sink in. Ask for code review; criticism is not a sign of weakness.

- Failure modes & gotchas
  - Misreading "define errors out of existence" as licence to ignore errors. Students skip network-failure handling and claim they "defined them away". Real failures still happen. Use it sparingly, "like a spice".
  - Outages often come from errors mapped wrongly (an error treated as success, or an unexpected response). Planning tends to be optimistic and skips the question "what goes wrong and how do we catch it?"
  - Pattern-picking (Gang of Four) is not design. It covers only a tiny fraction of real design work.
  - Relying on AI instead of comments: ChatGPT helped him get through the under-commented Linux kernel, but a few comments would have been far cheaper. Don't use AI as an excuse to skip comments.

- Relevance to Sam
  - Supplier integrations (Viator/RateHawk/Coras): wrap each provider in a *deep* adapter with a small interface such as `search`, `quote`, `book`, `cancel`. Hide auth, pagination, retries, rate limits and response normalisation inside it. Collapse provider-specific errors into the few categories the booking flow actually acts on (e.g. retryable / not available / our bug). Never map an unknown response to success.
  - Data scripts growing into products: resist splitting a pipeline into dozens of tiny entangled helpers. Group by stage (load / clean / match / write) with clear interfaces and interface docstrings.
  - Working with AI: AI writes the tactical code, so Sam's leverage is the decomposition and interface choices. Ask the AI for two different designs before it writes code. Review AI output for shallow modules and over-split functions. Keep docstrings on interfaces.

## Ned Batchelder — Getting Started Testing, PyCon 2014 (https://www.youtube.com/watch?v=FxSsnHeWQBY)
**Tests are the engineering answer to a real problem, "I don't know whether my code works". Treat test code as real code, and isolate what you test.**

- Mental models / first principles
  - Testing is not paperwork. It turns the fear of changing code into boredom. Hard work, but it pays back in time saved, more modular code and confidence when refactoring.
  - He grows tests in steps to show *why* frameworks exist: poking in the REPL (not repeatable) → a script that prints values (repeatable but you check by eye) → printing expected values next to actual (still manual) → `assert` (automatic, but the first failure stops everything) → you actually need independent, isolated checks, which is what `unittest`/pytest provide.
  - Five properties of good tests: **automated** (one command), **fast** (or nobody runs them), **reliable** (if you doubt a test, you've only moved the question from "does my code work" to "do my tests work"), **informative** on failure (clues for debugging), **focused** (each test runs as little code as possible, the opposite of product code, so a failure points somewhere specific).

- Mechanisms
  - `unittest`: test classes subclass `TestCase`, test methods start with `test`, run with `python -m unittest module`. The runner creates a *fresh TestCase instance per test method* and wraps each in try/except. That is how you get isolation: a failure can't stop other tests, and state can't leak between them.
  - Three outcomes: `.` pass, `F` (an assertion failed: the code is wrong), `E` (some other exception: the *test itself* is broken). Tracebacks print at the end.
  - Prefer `assertEqual(a, b)` over a bare `assert a == b` in unittest so the failure shows both values. (note: pytest's assertion rewriting gives this for bare `assert` too.) Other helpers: `assertIn`, `assertAlmostEqual` for floats, regex matchers.
  - Negative tests: `with self.assertRaises(ValueError): p.sell("IBM", 1000)`. Just calling the code would show up as an E, not a pass.
  - `setUp` runs before every test (build a fresh fixture such as a portfolio). `tearDown` cleans up afterwards (files, patches). `addCleanup` is the nicer option. For large shared data, look into fixtures.
  - Domain-specific base class: `PortfolioTestCase` with `assertCostEqual(p, 0)`. This lifts tests into the language of your domain. Web example: one helper that checks both status 200 and that a word appears in the body.
  - Test doubles for dependencies (slow, unpredictable or unavailable services, like the live Yahoo price feed):
    1. Monkeypatch a method on the object (`p.current_prices = fake`). Predictable, but coverage shows the real network/parse lines never run.
    2. Fake the lower layer (`urllib.urlopen`) so it returns canned CSV. Now the parsing code runs too and coverage is 100%, but it took hand-written plumbing and you must restore it in tearDown.
    3. `mock.patch("module.urllib.urlopen")`, set `.return_value`, then `assert_called_with(url)`, which also checks the call arguments (a hand fake silently accepts a broken URL).
  - Coverage (`coverage.py`) tests your tests: it shows which lines never ran.

- Decisions & trade-offs
  - Mocks cut dependencies, add speed and determinism, but they are **fragile and coupled to implementation**. The mock works only because you know the code calls `urllib.urlopen`. Switch to `requests` and every test breaks, and worse, the real network gets hit. If the real API changes format (CSV → JSON), the mocks still return the old format: tests pass, production fails.
  - Dependency injection (pass the HTTP client in at construction, defaulting to the real one) avoids patching module internals by name. Cost: cluttered signatures, and hundreds of injected dependencies don't scale.
  - DRY applies to tests: repeating the same 10 lines of setup across 60 tests is silly. Refactor into helpers and base classes. Some people want each test fully self-contained as documentation; he finds that weak.
  - Where reusable test data goes (setUp vs per-test vs fixture files) has no universal answer. It's an engineering decision like any other.

- Rules of thumb
  - Each test must pass when run alone or in any order.
  - Write tests for your test helpers once they become significant. That's a sign of maturity, not waste.
  - Well-tested systems often have more test code than product code. That's normal.
  - Keep `if __name__ == "__main__": sys.exit(main(sys.argv))` to one line. Make `main(argv) -> status` a plain function you can test. All I/O wiring sits in that one untested line. (note: with argparse, pass `sys.argv[1:]` or default `argv=None` and call `parser.parse_args(argv)`, otherwise the program name is parsed as an argument.)
  - Never edit product code (stubbing in `pass`) to run tests. Tests should reconfigure the system around the code under test automatically, with one button.
  - Tools: pytest (better runner), ddt or parametrisation for data-driven tests, coverage, CI on every push.
  - Lesson from years of practice: tests get complex faster than the system does, so invest in test structure early. And he learned the downsides of mocking the hard way.

- Failure modes
  - An `E` instead of an `F` means the test is broken.
  - Mocks drifting from the real interface give false confidence.
  - Relying on side effects between tests makes it impossible to know what any single test proves.

- Relevance to Sam
  - Provider API clients (Viator/RateHawk): don't patch `requests.get` deep inside the code. Inject a small HTTP client or session into the adapter, and test the *parsing and normalisation* against recorded real responses (saved JSON fixtures). That keeps coverage of the parse code without hitting the network. Refresh the fixtures when the provider changes, because mocked-format drift is exactly the failure Ned describes.
  - Data pipelines: make transforms pure functions over small DataFrames or rows so tests stay focused and fast. Keep a thin `main(argv)` for the CLI.
  - Domain assert helpers (`assert_location_matches(row, geonameid)`) keep pipeline tests readable as the project grows.

## Raymond Hettinger — Beyond PEP 8: Best practices for beautiful intelligible code, PyCon 2015 (https://www.youtube.com/watch?v=wf-BqAjZb8M)
**PEP 8 compliance is surface polish. The "gorilla" reviewers miss is non-Pythonic design. Fix it by wrapping bad APIs in Pythonic adapters so business logic reads clearly.**

- Mental models / first principles
  - PEP 8 is a *guide*, not a law book. Its own opening warns that a foolish consistency is the hobgoblin of little minds. "PEP 8 unto thyself, not unto others": the author tidies their own code; nobody uses it as a weapon on someone else's.
  - Attention problem (Stroop test, invisible-gorilla video): focusing on formatting makes reviewers blind to what matters. Fully PEP 8-compliant code can still be bad, and it gets copied as a template.
  - "P vs NP" = Pythonic vs Non-Pythonic. Pythonic code works with the language's own protocols (sequences, iteration, context managers, properties, magic methods), so the code that remains reads like business logic.
  - Low-hanging-fruit trap: struggling team members gravitate to PEP 8 reformatting because it feels productive. It churns `git blame`, scrambles the original authors' code, and reformatting patches to CPython have surprisingly often introduced bugs.

- Mechanisms (the routing-table case study, from real client code)
  - The original: a Java-generated SDK binding with `get_routing_table()`, `get_size()`, `get_route_by_index(i)`, `get_name()`, deep dotted imports, and try/except/else plus rollback/commit/disconnect wrapped around every use. Formatting it to PEP 8 made it prettier but no better.
  - Fix: an **adapter class** (about 5–10 minutes of work) that changes the shape of the API without doing any business logic, like a power-plug adapter:
    - A one-line custom exception (`class NetworkElementError(Exception): pass`). Catch the SDK's uninformative exception and re-raise a clearly named one with a clear message. "A great reason to wrap is to replace crummy error messages."
    - `__enter__`/`__exit__`: on error, log and roll back; otherwise commit; always disconnect. Setup and teardown are written once, and every caller uses `with NetworkElement(ip) as ne:`.
    - `@property routing_table` instead of `get_routing_table()`; `route.name` instead of `get_name()`.
    - `__len__` instead of `get_size()`, and `__getitem__` raising `IndexError` instead of `get_by_index`. Together these make the object a sequence, so `for route in ne.routing_table:` just works.
    - `__repr__` built from `type(self).__name__` (don't hard-code the class name, so subclasses get a correct repr).
    - A single flat module re-exporting what callers need, instead of deep dotted package imports. Packages exist to resolve namespace conflicts in very large codebases; most code doesn't need them.
  - Result: the calling code shrinks to roughly `with ... as ne: for route in ne.routing_table: print(route.name, route.ipaddr)`. Error handling has been pushed down into the adapter. He reports cutting client codebases from 50k to 10k lines this way.
  - Getters and setters are a smell in Python. Java needs them because you can't later swap a field for computed access; Python can add a `property` afterwards without breaking callers. Use plain attributes and promote them to properties when needed, even when each access triggers an underlying API call.

- Decisions & trade-offs
  - Line length: about 90 characters ("ish") on your own team, outside the stdlib. Strict 79 pushes people into atrocities like shortening names or awkward line breaks (and, reportedly, Google's 2-space indents). The limit should be a nudge that you're putting too much on one line, not a hard rule.
  - Names matter *more* in Python than in C. C has a type declaration next to the variable; Python doesn't, so names carry the information. Parameter names are part of the public API because callers can pass them by keyword (e.g. `symbols`, not `args`).
  - `eval` (behind namedtuple): powerful and dangerous. Be careful with untrusted input, but don't ban it outright.
  - Quotes: copy what the REPL does (single quotes). Double quotes are coming back because of embedded JSON. Minor either way.
  - Alphabetise imports only when you have dozens.

- Rules of thumb (his "recipe for making code better")
  - Wrap ugly third-party or generated APIs in an adapter. Never let their style leak into business logic.
  - Recurring setup/teardown → a context manager.
  - Index/size/lookup methods → `__len__`/`__getitem__` (iteration comes free).
  - getX → property.
  - Uninformative exceptions → custom, well-named exceptions.
  - Always add a `__repr__`.
  - Use keyword arguments at call sites wherever a bare literal is unclear: `twitter_search('obama', numtweets=20, retweets=False, unicode=True)`.
  - Unlabelled tuples → `namedtuple` (`Color(hue=170, saturation=0.1, luminosity=0.6)`). About 10 seconds of work, self-documenting, with a free readable repr.
  - Clear function names: Python is readable only "if you meet it halfway".
  - Trailing comments should add information, not repeat the code.

- Failure modes
  - Reformatting someone else's code: blame churn and new bugs.
  - Treating a linter pass as code-review approval.
  - Copying a non-Pythonic but compliant script as the template for many more.

- Relevance to Sam
  - Provider SDKs and REST APIs (Viator, RateHawk, Coras) are the "Java-generated API" case. Build a thin adapter per provider with a context-managed session (auth/close), properties, iterable result collections (pagination hidden in `__iter__`), named exceptions (`SupplierUnavailable`, `RateLimited`), and `__repr__` for debugging. Business and booking logic stays clean.
  - Pipelines over millions of rows: use namedtuples or dataclasses for records instead of positional tuples, and keyword arguments for config-heavy calls.
  - AI-written code: let a formatter (black/ruff) handle PEP 8 automatically. Spend review attention on API shape, naming and error semantics, which is the gorilla.

## Sandi Metz — All the Little Things, RailsConf 2014 (https://www.youtube.com/watch?v=8bZh5LMaSmE)
**With tests at your back, refactor a big conditional into small methods and then small polymorphic objects. Duplication beats the wrong abstraction. "Make the change easy, then make the easy change."**

- Mental models / first principles
  - Her one-line advice: make smaller things (classes and methods) that know as little about each other as possible.
  - Code gravitates toward bigger: people add new code next to similar code, and if that place is an `if`, they add another branch. Past a tipping point, nobody would ever put a 10-line helper class beside a 5,000-line model. Following a bad pattern makes the mess worse. Sometimes you have to *create a new pattern* (refactor) before adding a feature.
  - Procedures have no seams. OO code sends messages between small objects, and each message is a seam where you can substitute a different object.
  - Open/Closed: arrange code so new behaviour means *adding* code, not editing existing code. Then tests stay green and there are no distant side effects.
  - DRY is a rule for novices because duplication is the only problem they can see. Experienced developers should tolerate duplication until the right abstraction shows itself. **"Duplication is far cheaper than the wrong abstraction."** Fixing the wrong abstraction by adding more complexity is like chasing a beach ball into an outgoing tide.
  - Requirements tell you how to rearrange code. Don't design for an imagined future; let the actual new requirement (add "Conjured" items) drive the refactor.

- Mechanisms (Gilded Rose kata, worked through)
  - Starting point: a `tick` method that is one 43-line `if` with 16 conditions, 3 magic strings and many magic numbers. Flog (an ABC metric: assignments, branches, conditions) scored 45 for the method and 50 for the class. Her attempt to add Conjured directly failed after hours: every change broke something else.
  - **Squint test**: lean back and squint. Changes in *shape* mean nested conditionals; changes in *colour* (syntax highlighting) mean mixed levels of abstraction.
  - Step 1, make a seam: a `case name` at the top of `tick` routes "normal" to a new `normal_tick` method and returns early. Confirm the expected tests now fail (proof you've captured that execution path).
  - Step 2: make each test pass one at a time with deliberately dumb code. **"Red is when you scramble toward green, not when you ponder the abstraction."**
  - Step 3, refactor on green: pull out common steps (always decrement days; return early if quality is 0; always subtract 1, plus 1 more past the sell-by date). The result is flat and at one level of abstraction.
  - Repeat for Brie, Sulfuras (an empty method passes, because the item never changes) and Backstage. Then **delete the original 43-line conditional without ever understanding it.** The tests are the safety net.
  - Small methods: Flog 40 for the class, but the worst method is only 12.
  - Small objects: methods with a shared suffix and different prefixes (`normal_tick`, `brie_tick`) mean "there's a tortured object trying to get out". Create classes `Normal`, `Brie`, `Backstage`, each with `tick`. GildedRose holds an item and forwards to it. Temporary ugliness (if item: item.quality else @quality) disappears once every type is an object.
  - GildedRose was now a Middleman (only forwarding messages). Its one real job is choosing the class, so it becomes a **factory**: `GildedRose.for(name, quality, days)` returns an item, and callers talk to the item. This separates *the reason you switch* from *what you do when you switch*.
  - Inheritance: a base `Item` holds the shared attributes and a default no-op `tick`. Sulfuras then needs no class at all; it maps to `Item`.
  - Pull the string→class mapping out of the case statement into a hash: **configuration, not logic**. It can later move to YAML or a database as it changes more often.
  - Conjured is then one small new class plus one entry in the hash. Final average complexity is about 7 per class (worst 12): roughly 75% less complexity to reason about at once, because you read one small class, not the whole thing.

- Decisions & trade-offs
  - Intermediate refactoring steps *raise* metrics (Flog went 50 up to about 33-and-higher intermediate states before falling). Without principles you'd stop at the scary middle; with principles you trust where you're going.
  - Inheritance is safe when the hierarchy is **shallow and narrow**, subclasses are leaf nodes in the object graph (they know about no other objects), and every subclass uses all of the superclass's behaviour. Otherwise, use composition.
  - Superclass default for a role method: either raise NotImplementedError or provide a sensible no-op default. She chose the no-op.
  - Metrics are fallible, but human opinion is no more precise. Use both.

- Rules of thumb
  - Refactoring means changing arrangement without changing behaviour. You need tests ("a wall at your back"). For legacy code you don't understand, write characterization tests around the edges, then refactor until you can delete the incomprehensible part.
  - Finish the current refactoring before chasing tangents (resist the DRY cleanup mid-stream).
  - Track duplication you're consciously tolerating with numbered "dup tags" (e.g. `# DUP-6`) so you can find every copy later.
  - Repeated prefixes or suffixes on method names suggest a missing object.
  - A `case` or `if` on a type code combined with behaviour means: a factory picks the class, polymorphism does the work.

- Failure modes
  - Bolting another branch onto a huge conditional.
  - Premature DRY creating the wrong abstraction.
  - Middleman classes that only forward.
  - Deep or wide inheritance, or subclasses that override most of the parent.
  - Skipped tests hiding requirements (the six skipped "Conjured" tests were the spec).

- Relevance to Sam
  - Supplier-specific branching (`if provider == 'viator': ... elif 'ratehawk'`) or per-transport-mode logic in GTFS/location pipelines: refactor into one class or function per variant behind a common role (`normalise(raw) -> Offer`), plus a registry dict `{name: Adapter}`. Adding a supplier is then a new class plus one registry line.
  - Legacy data scripts: before reworking a messy `locations_transform.py`-style script, capture current outputs on a sample as golden/characterization tests, then refactor freely.
  - AI-generated code tends to over-DRY into premature helper abstractions *and* pile branches into one function. Ask for the dumb duplicated version first, then abstract once three real cases exist.

## Arjan Egges (ArjanCodes) — Stop Hardcoding Everything: Use Dependency Injection (https://www.youtube.com/watch?v=Xhzn1eAxoXk)
**Pass a component's collaborators in from outside instead of creating them inside it. Wire everything in `main`. Reach for containers or frameworks only when you need them.**

- Mental models / first principles
  - Problem: a `DataPipeline` class that hard-codes CSV loading, cleaning and JSON export as its own methods. Changing the source (another file type, a DB) or the sink (a DB instead of JSON) means editing the class. It's fragile, hard to extend, and hard to test, because once constructed all the behaviour is baked in.
  - Dependency injection: whoever *uses* a dependency should not *create* it. Benefits: flexibility (swap implementations), testability (swap in fakes), separation of concerns.
  - Dependencies don't always look like objects in the original code. Here they're methods on the class (`self.load_data_from_csv`) that should become injectable.

- Mechanisms (the progression)
  1. Pull the methods out into plain module-level functions (`load_data_from_csv`, `clean_data`, `export_to_json`). Already better, but `run` still hard-codes which functions it calls.
  2. **Functional DI**: `run(loader: Callable[[], Data])`, which makes `run` a higher-order function. Pass `load_data_from_csv` in. Arjan would often go further and make the whole pipeline a function.
  3. **Class-based DI**: `InMemoryLoader.load()`, `CleanMissingFields.transform(data)`, `JSONExporter(filename).export(data)`. `DataPipeline.__init__(loader, transformer, exporter)` stores them, and `run()` is simply `exporter.export(transformer.transform(loader.load()))`. `main()` builds the concrete objects and assembles the pipeline (the composition root).
  4. **Abstract with `typing.Protocol`**: `DataLoader` (load), `Transformer` (transform), `Exporter` (export). The pipeline depends on the protocols only. Key point: protocols specify *method* behaviour, **not `__init__`**, so a JSON exporter can take a filename while a network-stream exporter takes something completely different. Python doesn't enforce these at runtime (type checkers do), but they force you to state exactly what the pipeline needs.
  5. New behaviour without touching the pipeline: write a different transformer class (his example filters out rows by name) and pass it in `main`.
  6. **A tiny DI container** to show how it works: `providers: dict[str, tuple[Callable[[], Any], bool]]` plus a `singletons` cache. `register(name, provider, singleton=False)`; `resolve(name)` returns a cached singleton if one exists, raises `ValueError` if no provider is registered, otherwise calls the provider and caches it if it's a singleton. Register parameterised objects with a lambda or `functools.partial` (`lambda: JSONExporter("out.json")`). The pipeline provider resolves its own dependencies.
  7. **FastAPI `Depends`**: `def get_transformer(): return CleanMissingFields()` and an endpoint parameter `transformer: Transformer = Depends(get_transformer)`. The framework handles injection, and tests can override the dependency with a mock easily. Typical uses: DB sessions and services.

- Decisions & trade-offs
  - Functions vs classes: for single-method dependencies, inject functions (callables). Use classes plus Protocols when dependencies have several methods or their own state.
  - Manual wiring vs container: a container *is* more complex than just constructing objects in `main`. Its only real gain is flexibility over *where and how* wiring happens, e.g. building a pipeline from a JSON/YAML config listing which loader, transformers and exporter to use. Arjan says that in about 99% of cases he wires things manually in `main`. He uses FastAPI's built-in DI when he's on FastAPI, and rarely libraries like `inject`.
  - When to use DI: testable business logic (inject, then replace with fakes), swapping implementations (local vs cloud storage), ML or data pipelines where you try out components, web endpoints independent of a specific DB connection.
  - When not to: small scripts. "Sometimes just calling a function is enough." Don't build containers you don't need.

- Rules of thumb
  - Depend on behaviour (a Protocol or Callable), not on concrete classes.
  - Keep construction (`main` / composition root) separate from use (pipeline logic).
  - Protocols define what you call, never how it's constructed.
  - Testing: inject a test loader that returns fixed rows and a fake exporter that records instead of writing files.

- Failure modes
  - Hard-coding I/O inside business logic makes it untestable without touching the filesystem or network.
  - Over-engineering a container for a script.
  - (Compare Ned Batchelder: DI avoids fragile monkeypatching of module internals, at the cost of signature clutter if you inject dozens of things.)

- Relevance to Sam
  - This maps almost directly onto his work. Location and GTFS pipelines can become `Loader` (CSV/Parquet/GeoNames dump chunks) → list of `Transformer`s (clean, dedupe, match) → `Exporter` (Parquet/Postgres). Wire them in `main` or from a small YAML config, so a new source or sink is a new class, not an edit.
  - Supplier integrations: inject the HTTP client or session and the clock into adapters. Tests pass a fake client that returns recorded JSON.
  - For chunked processing on 16GB, Protocol-typed loaders can be generators (`Iterator[DataFrame]`) without the pipeline knowing.
  - FastAPI backends: use `Depends` for DB sessions and supplier clients, and override dependencies in tests.

## Casey Muratori (Molly Rocket) — "Clean" Code, Horrible Performance (https://www.youtube.com/watch?v=tD5NrevFtbU)
**Measured on Clean Code's own shape example, the rules that affect runtime (polymorphism over switches, hide internals, tiny single-purpose functions) cost 1.5x to 15x performance, 20x+ against lightly optimised code. Organise code by operation and data, not by type hierarchy, when performance matters.**

- Mental models / first principles
  - Most "clean code" rules can't be measured. But five of them change what actually runs, so they *can* be measured: (1) prefer polymorphism over if/switch, (2) never know an object's internals, (3) functions should be small, (4) functions should do one thing, (5) DRY.
  - The hardware is very fast and software is slow. For a class of programs, he argues, the abstraction style itself is the cause.
  - Grouping code by *function* (all the area formulas side by side in one switch) shows patterns that grouping by *class* (each formula in a separate file) hides. Once you see the pattern, you can collapse it into data.

- Mechanisms (C++ benchmark, cycles per shape)
  - Clean version: `Shape` base class with a virtual `Area()`, subclasses Square/Rectangle/Triangle/Circle. The array must hold *pointers* (objects differ in size), so every element is an indirection plus a vtable call the compiler can't see through. About **35 cycles/shape** (also tested hand-unrolled with 4 accumulators to break the loop-carried dependency, and with cold vs warm cache; little difference).
  - Switch version: one plain struct with an enum type plus width/height, stored *inline* in the array (all the same size), with `switch(type)` for the area. About **24 cycles: 1.5x faster**. He compares that to going from an iPhone 14 Pro Max back to an iPhone 11 Pro Max.
  - Table-driven version (breaks "don't know internals"): every area is `coefficient * width * height` (store height = width for squares and circles). Coefficients go in a `const float CTable[Count] = {1, 1, 0.5, Pi}`. Area = `CTable[type] * w * h`. About **3.5 cycles: about 10x faster**, which he equates to roughly 12 years of desktop CPU progress.
  - Add a second property (corner count; compute area weighted by `1/(1+corners)`): the clean version needs two virtual calls per shape, and clean-code style forbids fusing them. The switch version is about 2x faster. The table version folds the weighting into the table and is about **15x faster**. *Adding requirements widens the gap*, because more virtual calls hide more from the compiler.
  - Lightly hand-written AVX versions (select-based and table-based) reach **20–24x** over the vtable version, and that data layout is incompatible with clean-code structure anyway.
  - DRY: he's fine with "don't write the same code twice", but rejects it if it would forbid keeping two tables with the same data when that's needed for performance.

- Decisions & trade-offs
  - His position: those four rules are "absolutely horrible for performance" and shouldn't be taught without a warning that code may get 15x+ slower. Maintainability and readability are fine goals, but these rules are the wrong way to reach them.
  - (note: This is a deliberately polemical, microbenchmark-scale argument about hot inner loops in compiled code. Ousterhout's "deep modules" are compatible with it: Casey's table-driven function is a deep, simple interface. Metz's polymorphic refactor is about *change-heavy business rules* with a handful of objects, where per-call overhead is irrelevant. The useful synthesis: put polymorphism at the *boundaries* (one call per supplier, per file, per batch) and data-oriented, branch-light code in the *hot loops* (per row).)

- Rules of thumb
  - Measure in cycles or time per item. Don't argue about speed without a benchmark.
  - In hot loops, prefer homogeneous, contiguous data (struct-of-values, no pointers to heterogeneous objects) so the compiler and CPU can see everything.
  - When the variants differ only by a constant, replace dispatch with a lookup table.
  - Fuse operations over the same data into one pass instead of composing many tiny per-element calls.
  - Look at variants side by side (switch or table) to spot shared formulas.

- Failure modes
  - Per-element virtual dispatch through pointers: indirection, unpredictable calls, no inlining or vectorisation.
  - Tiny single-property methods called separately per element block fusion.
  - Treating "clean" rules as free of cost.

- Relevance to Sam (Python translation; my extrapolation, not Casey's)
  - In Python the same principle shows up far more strongly. Calling a method on a per-row object (or `df.apply(lambda row: obj.method(row))`) over 6M GeoNames rows is interpreter-bound. Expressing the work as column operations over contiguous arrays (pandas/NumPy/Polars/DuckDB vectorised expressions) is often 10–100x faster.
  - Casey's table trick maps to `coef = np.array([...])[type_codes] * w * h`, or `Series.map(dict)`, or a join against a small lookup table, instead of `if/elif` per row.
  - Category columns as small integer codes (pandas `category`, Polars `Categorical`) also save RAM on a 16GB laptop.
  - Keep the clean, polymorphic adapter layer for *per-supplier* or *per-file* decisions (few calls). Inside each transformer, operate on whole chunks and columns.
  - When AI writes pipeline code, check whether it has produced row-wise loops or `.apply` with objects; ask for a vectorised version and benchmark both on a sample.

## James Murphy (mCoding) — 25 nooby Python habits you need to ditch (https://www.youtube.com/watch?v=qUeud6DvOWI)
**A checklist of idiomatic Python. Some items are real bugs (mutable defaults, bare except, unclosed files, shell=True); the rest signal inexperience and cost readability.**

- Real correctness and safety issues
  - **Mutable default arguments**: defaults are evaluated *once, when the function is defined*, so every call shares the same list or dict. Use `def f(x, acc=None): if acc is None: acc = []`.
  - **Closing files manually**: if `write` raises, `close()` never runs. Use `with open(...) as f:`. More generally, use the resource's context manager instead of `try/finally` (a habit from other languages).
  - **Bare `except:`** also catches `KeyboardInterrupt` and `SystemExit`, so Ctrl-C gets trapped. At minimum use `except Exception:`; better, catch the specific exception you expect.
  - **`subprocess(..., shell=True)`** is a common source of security holes, usually used just to avoid writing the args as a list. Pass a list.
  - **`type(x) == T`** breaks substitutability (Liskov): a namedtuple *is* a tuple but not *literally* `tuple`. Use `isinstance`.
  - `^` is XOR, not exponent (`**`).
  - Relying on a flat directory layout: imports work only because Python adds the running script's directory to `sys.path`, and they break when scripts live elsewhere. Package your code and install it into the environment (e.g. `pip install -e .`).
  - `import *` outside the REPL pollutes the namespace. Import names explicitly.

- Idioms
  - f-strings instead of `+` concatenation.
  - Comprehensions (list, dict, set, generator) where they're clearer. But **don't turn every loop into one**: complex comprehensions read worse than loops.
  - `is None` / `is True` instead of `==` (identity is what's meant, and it skips the `__eq__` middleman).
  - `if x:` instead of `if bool(x):` / `if len(x):`. (note: not for NumPy arrays or pandas objects, where truthiness raises. Use `.size`, `.empty`, `len()` there.)
  - Iterate directly over the container, not `range(len(...))`. Use `enumerate` when you need the index, `zip` to walk two sequences in step, and `enumerate(zip(...))` for both.
  - Iterate over a dict directly, not `.keys()`. Use `.items()` when you need the values. Copy the keys (`list(d)`) only when mutating while iterating.
  - Unpack tuples: `x, y, z = point`.
  - No hand-rolled `i += 1` counters; use `enumerate`.
  - Time with `time.perf_counter()` (differences), not `time.time()` (wall clock, less precise).
  - Use `logging` instead of scattered prints: configure the format and level in `main`, and make the level a parameter so you can filter.
  - Use NumPy/pandas for numeric and data work rather than pure-Python loops.
  - Python *is* compiled, to bytecode (`.pyc` in `__pycache__`), which the interpreter then runs.
  - Follow PEP 8 mostly so people stop nagging. (Compare Hettinger: let a formatter do it and focus on what matters.)
  - Python 3 only. `range` is lazy and `x in range(...)` is O(1) via bounds. `dict.keys()` returns a live *view*, not a copy.

- Relevance to Sam
  - On long pipeline runs: use `logging` with timestamps and levels (and a `--log-level` flag) instead of print, and `perf_counter` for per-stage timings.
  - Mutable defaults and bare excepts are classic AI-generated-code bugs. In supplier clients especially, a bare `except` around HTTP calls can hide auth failures and swallow Ctrl-C during long runs. Make these lint rules (ruff: B006 mutable defaults, E722 bare except, S602 shell=True).
  - Turn growing script folders into an installable package (`pyproject.toml`, `src/` layout) as soon as a second script imports the first.

