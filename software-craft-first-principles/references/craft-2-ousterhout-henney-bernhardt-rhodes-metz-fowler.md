# craft-2 — cross-cutting principles

1. **Complexity is the enemy, and it builds up through hundreds of small compromises, not one big mistake.** Fight it continuously, with roughly 10–20% of effort, not with heroic rewrites. (Ousterhout, strategic vs tactical; Fowler, litter-pickup and design stamina)
2. **Deep modules: small interface, lots hidden.** The interface is everything a caller must know, including side effects and ordering. Judge a function or class by depth, not length. Hundreds of lines behind a simple interface are fine, and "max N lines" rules breed classitis. (Ousterhout)
3. **Make the common case trivial and push options to the rare case.** Buffering should be on by default, not a third wrapper object. (Ousterhout, Java I/O)
4. **Define errors out of existence.** Choose semantics where the "error" is normal behaviour: idempotent delete/unset, substring returning the overlap, delete-while-open. Throw only when you truly can't meet the contract. Exceptions pay off when they travel far up the stack. (Ousterhout)
5. **Hoist I/O, don't hide it.** Pull logic out into pure functions that take and return data, and leave I/O as thin glue at the top. Hiding I/O in a helper still couples every caller to it. (Rhodes; Bernhardt)
6. **Functional core, imperative shell.** The core makes the decisions (many paths, no dependencies), so give it lots of fast unit tests with no mocks. The shell holds the dependencies (few paths), so give it a few integration tests. "80% functional beats 0%." (Bernhardt; Rhodes)
7. **Values are the best boundaries.** Plain data passed between components can be tested without mocks, cached, serialised, put on a queue and parallelised. Values let you move boundaries between classes, processes and machines. (Bernhardt, actors; Rhodes, distribution)
8. **Prefer code you can picture as data flowing through stages.** Brooks: show me your tables. McIlroy's pipeline beat Knuth's 10 pages because each intermediate result is visible and checkable. Use comprehensions, `sorted()`, and generators over in-place mutation. (Rhodes)
9. **Mocks hide boundary bugs, and integrated tests explode** (2^n paths, superlinear runtime). The way out is design: pure cores need no mocks. When tests need heavy setup, many mocks, giant multi-assert bodies, or peeks at private state, the test is reporting a design problem. (Bernhardt; Farley)
10. **Tests are specifications.** Name them as propositions ("an empty stack throws when popped"), one case or behaviour each, asserting behaviour not implementation. Never add production APIs only for tests. Coverage targets breed tests that assert nothing. (Henney; Farley)
11. **Replace type checks and nil checks with objects.** The Null Object ("active nothing") removes scattered `if x is None`. Keep the single remaining conditional in a wrapper around the untrusted API. (Metz)
12. **Inheritance is for specialisation, not code sharing.** When variants multiply, find what varies, name the *role*, define its API, and inject players. The default is a player too, and there is never just one specialisation. Composition plus DI gives N×M behaviours with less code. (Metz)
13. **Make things more alike to reveal how they differ.** Refactor variants until only the differing piece remains, then name that column. (Metz)
14. **Name in the language of the domain.** Avoid Lego names (`...ManagerFactory`), redundant suffixes (`...Exception`), filler words (Object, Data, Info, Manager), stringly-typed primitives, and reflexive get/set pairs. Extract Method to name intent: `trader.can_view(portfolio)`. (Henney)
15. **Encapsulate state, don't expose representation.** No public mutable fields or setters "just in case", and don't return internal collections. Expose domain operations instead. (Henney)
16. **Comments are often noise written by the author of the unclear code.** Turn knowledge into names and extracted functions. Keep comments for what code can't say. (Henney; Rhodes citing XP; Fowler, comprehension refactoring)
17. **Two hats: change behaviour *or* restructure, never both at once.** Take small steps with tests after each, and undo when red. Use preparatory refactoring ("make the change easy, then make the easy change") before features, and comprehension refactoring right after you understand confusing code. (Fowler)
18. **Large redesigns happen by agreeing a target and moving toward it every time you touch the area**, keeping the code releasable. Planned "refactoring tasks" are a smell. (Fowler)
19. **Justify cleanup with economics, not professionalism.** Refactor what slows you down repeatedly, and leave working code nobody touches. (Fowler; Ousterhout's investment curve)
20. **Formatting and conventions should serve the reader and survive maintenance.** Use narrow lines, keep one concept looking like one visual unit, and avoid alignment that breaks on rename. Let a formatter decide. (Henney)

**Applied to Sam's work (synthesis):** pipelines as read → pure transforms (generators, chunks) → write, with intermediate Parquet you can inspect. One deep adapter module per supplier with pure `build_request`/`parse_response` functions tested on recorded JSON. Providers injected as role players (Protocol/callable) instead of `if provider ==` chains or subclass trees. Sentinels and null objects instead of None checks scattered through code. Review every AI diff for shallow wrappers, noise comments, Lego names, mixed feature-plus-refactor changes, and liar/mockery tests.

---

## John Ousterhout — A Philosophy of Software Design, Talks at Google (https://www.youtube.com/watch?v=bmSAYlu0NcY)
**Core idea in one line: design is the fight against complexity. Win it with deep modules (simple interface, lots hidden), by defining errors out of existence, and by working strategically: spend about 10–20% of your time on design, all the time.**

- **Mental models / first principles**
  - The most important idea in CS is *problem decomposition*: splitting a complex system into pieces you can build fairly independently. Hardly anyone teaches it explicitly.
  - Design skill can be learned. It comes from deliberate practice plus critique, the way writing is taught (write, get marked up, rewrite). His Stanford CS190 course runs three loops: build roughly 2–3k LOC in pairs, get heavy code review (50–100 comments per team), revise, then start a fresh project.
  - Complexity is incremental. It is not one big mistake but hundreds or thousands of small ones made by many people over time. So you don't notice it piling up, and once it's there you can't fix it with one change.
  - **Module as rectangle:** the area is the functionality (benefit) and the top edge is the interface (cost). The interface means *everything a caller must keep in mind*: signatures, side effects, dependencies, ordering rules. You want a small top and a big area. This is Parnas's information hiding (early-1970s paper, still worth reading) restated.
  - **Red flags** help beginners. Even if you can't see the right design, you can spot that you're going wrong, change approach, and repeat until the flags go away.

- **Mechanisms / examples**
  - Shallow method: invoking it costs more keystrokes than writing its body inline. No information is hidden and the result is a net loss.
  - Shallow class: a thin list-manipulation wrapper over two pointers that everyone knows are there anyway.
  - Classitis, with Java I/O as the example: to read serialized objects from a file you build three objects (FileInputStream, then BufferedInputStream, then ObjectInputStream), and each one can throw. Buffering should be the default. The *common case must be simple*, and the rare case is where the extra complexity belongs.
  - Deep interface example: Unix file I/O. Five calls (open/read/write/lseek/close) hide hundreds of thousands of lines: disk allocation, caching, device drivers. Before Unix, systems had separate call sets for random-access and sequential files.
  - **Define errors out of existence:**
    - Tcl `unset` threw when the variable didn't exist, so in practice almost every call got wrapped in `catch`. A better meaning is "make sure this variable doesn't exist", which makes the operation idempotent and removes the error.
    - Unix file delete while the file is open: the name is removed at once and the data lives until the last close. Neither side sees an error. Windows refused the delete, and its later flag-based fix still locked the directory entry, so a rebuild couldn't recreate the file.
    - Substring with out-of-range indices: return the overlap (possibly empty) instead of throwing. This removes clipping code from every caller.
  - In RAMCloud, about 90% of effort went into crash recovery, not the latency work they expected. Error handling is a major source of complexity.

- **Decisions & trade-offs**
  - Throw only when you genuinely cannot meet the contract (for example, a real I/O error on read). Don't throw for situations you can turn into normal behaviour. Some errors really do matter, and design is largely about deciding *what matters* and making as little matter as possible.
  - Exceptions vs return codes: exceptions pay off most when they travel far up the stack and skip many intermediate layers. If the immediate caller handles it, a return value is equivalent and often syntactically simpler.
  - Sometimes crashing is the right answer. Out-of-memory recovery, for example, is hopelessly complex, so print a message and die.
  - "Prevent misuse" APIs (like a substring that throws to catch caller bugs) usually add complexity for every caller. Make the right thing easy and rely on tests to catch mistakes.
  - Layers vs performance: the common mistake is many thin layers. Prefer fewer, meatier layers. If you know your key performance metrics at design time, you can usually draw the layers so they don't cost much speed.
  - Generality: making a class *slightly* general-purpose, even if it has only one caller today, often makes it simpler and deeper. He changed his own mind on this after seeing student projects.
  - Thinking ahead: design is for the future, but you can't see the future well, so don't design too far ahead.

- **Rules of thumb**
  - Length is not the issue. A method of hundreds of lines is fine if it's clean behind a deep interface. Rules like "split anything over N lines" (N = 10 or 20) cause classitis. Get depth first, and split only if the result is still unwieldy.
  - **Strategic vs tactical programming.** Tactical means "get it working" with a few kludges, and those kludges compound across the team. Strategic means the goal is a great design and working code is just table stakes. You go slower at first and faster later. His guess (no data) is that the payback crossover comes at 6–12 months, roughly how long it takes to forget why you wrote the code.
  - Invest around 10–20% extra in small, steady steps, not big up-front design. When you build a new module, spend a little time on the interface, write docs as you go, and write unit tests.
  - Every time you touch code, leave something better. Your change probably made something worse, so you need an improvement just to break even. Don't aim for the *fewest lines changed* (like reaching into another file's global). Aim for "what would this look like if I'd built it from scratch knowing what I know now?"
  - Zero tolerance for small lint, because once you let it in you slide back into tactical mode.
  - Hire for slope (learning rate), not y-intercept (current experience).

- **Failure modes**
  - The **tactical tornado**: a very prolific producer of code that 80% works, often treated as a hero by management, who leaves damage behind. Startups that are 100% tactical (Facebook's "move fast and break things") end up with spaghetti that is almost never cleaned up, short of a rewrite.
  - Students who took "define errors out of existence" too far wrote no handlers at all. Real errors still need handling.
  - Hyrum's law: with enough users, every observable behaviour becomes depended on, so be careful what an API exposes.

- **Relevance to Sam**
  - For a solo builder whose AI writes the code: AI defaults to tactical output and to classitis (many tiny wrappers). Review AI output for depth. Ask whether each function or class hides something real, or whether its caller still has to know everything.
  - Supplier integrations (Viator, RateHawk and others) are good candidates for one deep module per provider with a tiny interface such as `search(...)`, `book(...)` and `cancel(...)`. Retries, auth, pagination, rate limits and field mapping stay inside.
  - Data scripts: make operations idempotent ("ensure output partition exists", "delete if present") so that reruns of a multi-million-row job are not error cases. Normalise odd inputs (empty strings, out-of-range values) in one place instead of throwing in many.
  - Spend 10–20% of each feature on cleanup as you go. Pipelines that grow into products are exactly where tactical kludges compound.

## Kevlin Henney — Seven Ineffective Coding Habits of Many Programmers (https://www.youtube.com/watch?v=SUIUZ09mnwM)
**Core idea in one line: code is communication. Most bad code comes from unexamined habits, not malice, so treat your own coding conventions as requirements with a purpose: "spend less time working out what code means, more time coding meaningfully."**

- **Mental models**
  - Habits are low-energy automation, which is also why they're blind spots. We copy them from colleagues and from IDE defaults without asking where they came from. Nobody decides in a meeting to write a 10,000-line class. Systematic habits produce it.
  - Write developer-facing "user stories" for your own practices: *as a programmer I want X from this style/architecture so that ...*. The answer Henney gives is less time decoding meaning and more time adding it.
  - Code layout is visual design (citing Daniel Higginbotham): arrangement, alignment and grouping carry meaning. A style should meet the reader's information needs with minimal conscious effort, not "what I prefer".

- **The seven habits (mechanisms and fixes)**
  1. **Noisy code** (low signal-to-noise).
     - Verbosity: the first version you write is rarely the best, so revise once you understand it. Henney rewrote left-pad much shorter, and his version actually works.
     - Needless dependencies for 10-line utilities. The left-pad incident: thousands depended on someone else's mediocre code.
     - Comments as noise: giant licence headers and "display the string" on hello world. Rob Pike (1989): clear code with good names should explain itself, the compiler doesn't check comments, and comments clutter. The key insight: *the person asked to comment unclear code is the person who wrote it*, so they will explain it with the same muddled thinking. Bring in another reader, or raise the meaning of the code itself.
  2. **Unsustainable spacing / line length.**
     - Humans read narrow columns, which is why magazines use them. You don't need 80 columns, but triple-digit widths (he found code reaching column 360) are too wide.
     - A parameter list should look like *one thing*. A style that leaves the first parameter on the method line and lines the rest up far to the right splits it into two places.
     - Alignment that depends on name length (lining up under the opening parenthesis) breaks on every rename. Renames happen often because first names are rarely the best, so the style decays into inconsistency. That makes it empirically unsustainable unless every tool on the team enforces it (Lisp culture is the exception).
     - Count your axes of alignment: almost one per line is visually busy. You can judge a layout objectively by how readable it is and how it survives maintenance.
  3. **Lego naming** (agglutination): `AbstractManagerControllerProxyFactorySingleton`.
     - Gluing words together dilutes meaning. Name the thing itself: a "ConditionChecker" hierarchy was really just `Condition` with `isTrue()`.
     - Kent Beck's warning: people will use your names in conversation for 20 years. Ban filler words like Object, Thing, Stuff, Component, Part, Manager, Entity, Item.
     - The `Exception` suffix is noise. Context (throws/catch, the base class) already says it's an exception. Strip the suffix and good names remain (`AccessViolation`, `ArgumentOutOfRange`, `IndexOutOfRange`), while bad ones show they never meant anything (`Arithmetic`, `Format`, `NullReference`; the real event is a *null dereference*).
     - "Omit needless words."
  4. **Under-abstraction** (more common than over-abstraction).
     - Philippe Calçado's trick: tag-cloud the identifiers in a codebase with comments and strings removed. If `String`, `int` and `List` dominate, you have a *stringly-typed* system with no domain vocabulary. A healthy cloud shows domain nouns (Printer, Paper, Product).
     - Dan North's "code in the language of the domain": a long `if` over portfolio/trader/containsKey is really `trader.canView(portfolio)`. Extract Method is under-used. It adds meaning by naming intent.
  5. **Unencapsulated state.**
     - The Singleton is the ultimate global. Public fields and auto-properties with get/set are barely better.
     - Public `balance` with a setter: once the balance becomes derived (from a transaction history or a cache) or has rules (no negatives), the setter turns into a monster.
     - Returning internal collections exposes your representation. "Don't invite the vampire in": once internals cross the boundary you can't take them back. Expose domain operations (`deposit`, `withdraw`), not data.
  6. **Getters and setters.**
     - `get` and `set` have the longest entries in the OED, so they're among the most ambiguous words available. In English, "get" is an imperative with side effects (get drunk, get money from an ATM), so a "get" that only queries misleads.
     - The opposite of set is unset/reset, not get. The get/set pairing is a false symmetry written out of habit, often IDE-generated with no test needing the setter ("IDEs let you do the wrong thing faster").
     - Drop the setter, and drop the `get` prefix too: `units()`, `currency()`. Prefer a functional, query-style interface.
  7. **Uncohesive tests.**
     - IDE-generated one-test-per-method (`testConstructor`, `testPush`, `testPop`) mirrors the structure, not the behaviour. `pop` has at least two cases (empty and non-empty), and you can't use `push` without the constructor anyway.
     - Name tests as propositions about behaviour: "a new stack is empty", "an empty stack throws when queried for its top", "an empty stack throws when popped". Passing tests then read as a specification, and failures read as the negation of a claim.
     - One test, one case. Nat Pryce and Steve Freeman: tests that *drive* development must express what "working" means, not just check that the code works.

- **Rules of thumb**
  - Revise after first drafts, especially names.
  - Before adding a dependency, ask whether it's 10 lines you could own.
  - Delete comments that restate code. When code needs a comment to explain it, first try a better name or Extract Method.
  - Choose a layout that survives automated renames and reformatting.
  - Use domain words, not technical suffixes.
  - Don't expose mutable internals. Don't add setters without a need.
  - Name tests as sentences describing behaviour, one case each.

- **Relevance to Sam**
  - AI assistants reproduce exactly these habits: verbose comments restating code, get/set pairs, `DataManager`/`ProcessorHelper` Lego names, and one test per method. Make the review checklist explicit: strip noise comments, rename to domain words (`Place`, `Supplier`, `Booking`, `Itinerary`), delete unused setters.
  - Pipelines that pass dicts of strings everywhere are stringly typed. Introducing small domain types (a dataclass `GeoPlace` or `Rate`, or enums for provider status) is the Python version of his under-abstraction fix.
  - Write pytest names like `test_unmatched_place_is_left_unlinked` rather than `test_match_function`.
  - Use a formatter (black/ruff) so layout is never a personal choice and survives renames. That is exactly his "sustainable" criterion.

## Gary Bernhardt — Boundaries, RubyConf 2012 (https://www.youtube.com/watch?v=yTkzNHF6rMs)
**Core idea in one line: make values the boundaries between components. Put all the decisions in a functional core of pure value-in/value-out code, and wrap it in a thin imperative shell that holds the dependencies and state. Unit-test the core heavily and the shell with a few integration tests.**

- **The problem that motivates it (testing)**
  - Example: a `Sweeper` that finds users who are active but haven't paid in a month, then emails them and disables them. An isolated unit test stubs `User.all` and mocks `UserMailer.billing_problem`.
  - Isolated tests have three benefits:
    1. Design feedback: if you need six mocks, some three calls deep, the design is bad.
    2. Outside-in TDD: build high-level pieces first; the stubs tell you what interfaces the lower pieces need.
    3. Speed: about 200 ms against about 30 s for even a tiny Rails test.
  - The big cost: tests run against fakes and production runs against real classes. A wrongly stubbed boundary gives a green test and broken production. For most people this outweighs the benefits.
  - Partial fixes:
    - Contract/collaboration tests (J.B. Rainsberger).
    - Verifying doubles (rspec-fire, which only lets you stub methods that really exist; in Python, `mock.create_autospec`/`spec_set`).
    - Static typing.
    - All three catch only *shallow* mismatches (wrong name, wrong arity), not "two algorithms that don't actually cooperate correctly".
  - **"Integrated tests are a scam"** (Rainsberger):
    - Paths grow as 2^n in the number of branches (every if, try/except, short-circuit, loop). 500 conditionals give a number with about 150 digits, so you can't choose the right paths to cover.
    - Integrated suite runtime grows superlinearly: each new test adds a test *and* makes the system bigger, which slows every existing test. Hence three-hour Rails suites.

- **Mental model: why `+` needs no mocks.** It isolates for free because (1) it takes values and returns *new* values without mutating anything, and (2) it has no dependencies. It isn't simplicity. Apply the same two constraints to real code:
  - The `Sweeper` test becomes plain user structs in, and a list of expired users out. The mailing half moves elsewhere.
  - The *nature of communication* changes: components talk through returned values instead of synchronous calls into each other.
  - A struct with a derived field (slug computed lazily vs at construction) is behaviourally the same value. If every method on an object is pure, the object *is* data.

- **Paradigm table** (mutation? × data and code bundled?):
  - Procedural: mutates, data and code separate (`walrus.stomach << food`, so you know the internals).
  - OO: mutates, bundled (`walrus.eat(food)`).
  - Functional: immutable, separate (`map` producing new walrus hashes).
  - "FauxO": immutable *and* bundled. `walrus.eat(food)` returns a *new* walrus. Objects with methods, but never mutated.

- **Functional core, imperative shell**
  - FauxO alone can't do I/O or keep state over time. So add a shell: fetch from the DB, call the core (`expired_users(users)`), then email each result.
  - The split isn't arbitrary: **the core makes all the decisions; the shell has all the dependencies.**
  - Core: many paths, few dependencies, which suits isolated unit tests with no mocks at all.
  - Shell: many dependencies, few paths, which suits a *small number* of integration tests. Example: create Alice (paid two months ago) and Bob (paid yesterday) in a real DB, run the sweeper, assert only Alice got mail. One test like that versus many core tests.
  - Real example, his roughly 600-line terminal Twitter client:
    - Pressing `j` makes the shell call `cursor.down`, which returns a new Cursor. The shell just reassigns `@cursor`.
    - The Cursor class (tweets plus selection, about 100 lines) was the key boundary choice. Keeping the tweet list and selection separately "would have been awful".
    - The 153-line shell had no tests at all. He admits that's not ideal.
  - Choosing core boundaries is the hard part: small, cohesive components, not three-liners and not whole subsystems. "Separate the things that mutate from the things that don't" is the best starting rule he has found.
  - Bonus: values are serialisable messages, so the same design converts to the actor model (threads plus queues as inboxes; his actor library was about 35 lines). Fetcher, filter and mailer become three processes on three cores with little rework. "Values afford shifting boundaries": between classes, subsystems, processes, and serial vs parallel.

- **Decisions & trade-offs**
  - Large systems: don't build one giant core and shell. Haskell IO-monad programs work well up to about 500 lines and then hurt. Build many small core/shell systems talking over channels, the way Erlang achieves very high availability. The shell is essentially the adapter ("scar tissue") in ports-and-adapters.
  - Database: don't actually do `User.all` and filter in Ruby. Let the DB filter; querying is a small part of overall complexity. Postgres/MySQL live in the shell; an immutable DB like Datomic could sit in the core. **"80% functional is a heck of a lot better than 0%."**
  - Frameworks: a 100k-line Rails app is really 95k lines of your app plus 5k of glue. Dumping it all into models and controllers means you never designed the system. Third-party libraries (gems) are fine; they live in the shell.
  - Performance: lots of allocation hurts on some VMs (MRI). Immutable values can be cached freely, which compensates.

- **Gotchas**
  - Mocking fixes the speed problem but creates boundary-mismatch risk. Pure-core design removes the need for mocks rather than making mocks safer.
  - Methods named `each` with side effects signal procedural mutation. Prefer `map`/`select` that return new values.

- **Relevance to Sam**
  - This is the single most useful pattern for his pipelines. Pure functions take rows or DataFrames/Arrow batches and return new ones: normalise names, match places, score candidate matches, map supplier fields to the internal model. The decisions live there and are unit-tested with tiny literal inputs and no mocks.
  - The shell is the thin `main()`/runner: read the Parquet/CSV chunk, call the core, write the output, call the supplier API, log.
  - Supplier integrations: `parse_viator_response(json) -> list[Product]` and `build_booking_request(cart) -> dict` are pure and tested against saved fixture JSON. The HTTP client, auth and retries form the shell, covered by a handful of integration or sandbox tests. That avoids the "mock passes, production breaks" trap.
  - Chunked or parallel processing on a 16GB laptop: if the core is value-in/value-out, moving to `multiprocessing`/`concurrent.futures` or queue-based workers is nearly free. That's his "values afford shifting process boundaries".
  - When AI writes code, ask for this split explicitly. Generated code tends to interleave I/O with logic, which makes it untestable without mocks.

## Brandon Rhodes — The Clean Architecture in Python, PyOhio 2014 (https://www.youtube.com/watch?v=DJtef410XaM)
**Core idea in one line: don't *hide* I/O in subroutines, *hoist* it. Pull the logic out into pure functions that take and return data, and leave the I/O stranded at the top as thin procedural glue ("make the I/O the manager").**

- **Mental model**
  - Since D.J. Wheeler's 1952 ACM subroutine paper, subroutines have sold themselves as a way to "bury complexity out of sight". The habitual mistake is to bury the messy I/O (HTTP, JSON, DB, files) in a helper. That *hides* it but doesn't *decouple* it: calling the higher function still inevitably does I/O, so you can't run it on cached data or in a test without the network.
  - The reverse move: rescue the logic, not the I/O. His three listings:
    - Listing 1 is one procedure that builds a URL, calls `requests.get`, and plucks the definition from the JSON.
    - Listing 2 moves the HTTP call into a helper. That's a failure, because the top level is still a procedure with I/O buried under it.
    - Listing 3 is the success: `build_url(word)` and `pluck_definition(data)` are pure, and the top-level function just glues `url = build_url(...)`, `data = requests.get(url).json()`, `return pluck_definition(data)`.
  - Uncle Bob's Clean Architecture (and the hexagonal architecture before it): the inner circles are policies (business rules), the outer circles are mechanisms (DB, web, frameworks). *Simple data structures cross the boundaries.* Swapping an obsolete DB or framework only touches the outside.
  - Naming the extracted pieces adds meaning, like XP's "a comment above a block of code is a function waiting to be extracted".

- **Testing consequences**
  - For I/O-coupled code there are two workarounds:
    - Dependency injection (Fowler, 2004): pass `requests` in as a parameter with a default. It spreads, because top-level functions end up needing a "kitchen sink" of injected things for whatever is deep underneath. That's how Java ended up with DI frameworks.
    - `mock.patch` (Michael Foord's mock, now `unittest.mock`): patch `requests.get` inside a `with` block.
  - Both feel like "fighting the structure of the app", and a fake isn't the real thing, so a passing test doesn't prove the real API works.
  - Pure functions are tested with data in and data out, no setup and no mocks. The test call looks exactly like the production call, so a passing test means more. `pluck_definition` reached 100% coverage with two tests.
  - **Coupling smell found through tests:** if testing step 2's failures means feeding step 1 good inputs while varying step 2's, and the tests break whenever you reorder steps, the steps are separate logic wrongly fused. Split them so each has its own small test set. Exception: if step 1 produces about 20 intermediate values that step 2 needs (common in astronomy code), splitting may not be worth it.
  - For the top-level glue, follow Gary Bernhardt: many fast unit tests on the functional core and a few end-to-end tests on the imperative shell, just to prove the wiring and order are right. Glue with no `if`s needs very few tests.

- **Why data-centric code is easier (his argument: data, not immutability, is the main win)**
  - Fred Brooks (1975): if you show your tables (data structures), the flowchart becomes obvious. The reverse doesn't hold.
  - McIlroy vs Knuth (1986): print the K most common words. Knuth wrote 10 pages of literate Pascal with a clever 27-way hashed trie. McIlroy wrote a 6-stage shell pipeline: `tr` non-letters to newlines, `tr` to lowercase, `sort`, `uniq -c`, `sort -rn`, `sed ${K}q`.
    - McIlroy's lesson: small composable tools that were generalised after first being written for one need.
    - Rhodes's extra lesson: you can *picture the data between every stage*. Intermediate results are visible and checkable, so you can find which step broke.
  - The point of immutability (crediting Bernhardt): distribution. If the return value is what matters, the step can run on any core or machine behind a queue. A mutate-in-place step must run where the data lives.

- **Pythonic mechanisms that push this way**
  - List comprehensions (2000) turned programmers from mutators into builders of new structures. Chain them like a pipeline of throwaway intermediate results.
  - `sorted()` (2.4) returns a value instead of `.sort()` mutating and returning None.
  - Python has three extraction tools where Java has one:
    - a function, for the body of a loop
    - a **generator**, for the iteration logic itself (a `upper_words(words)` generator plus a top-level loop that prints, instead of a function that both transforms and prints)
    - a **context manager**, for setup and teardown
  - Prefer functions over methods for logic: a method implicitly depends on the whole object's state, so it's unclear what to set up in a test. A function's argument list states exactly what it needs ("explicit is better than implicit").

- **Real examples**
  - Skyfield (astronomy): an object API backed by dozens of pure, easily tested functions.
  - Luca (his tax-form filler): v1 wrote each PDF field as soon as it was computed, which was hard to maintain. It was rewritten as three phases: read all inputs into a data structure, compute every output line in pure routines (add, round, apply percentages), then hand the result to a PDF writer.

- **Rules of thumb**
  - Ask of every function: can I call this without I/O happening? If not, extract the logic below it.
  - The I/O-bearing code should be the *top* of the call graph, not the leaves.
  - Pass plain data across boundaries.
  - Structure programs as phases: read everything, transform, write.
  - If testing different failure branches means contorting inputs, split the function.

- **Relevance to Sam**
  - This maps directly onto his pipelines: `read_chunk()` → `normalise(rows)` → `match(rows, index)` → `write(rows)`. The middle stages are pure and tested on 5-row literal fixtures. You can also dump intermediate Parquet files between stages, which gives McIlroy-style "see the data at each step" debugging on million-row runs.
  - Generators let a 6M-row GeoNames job stream through pure transforms in bounded memory on 16GB.
  - Supplier clients: `build_search_params(query) -> dict`, `parse_rates(json) -> list[Rate]`. The `requests`/`httpx` call is the only impure line. Test parsers against recorded real responses rather than `mock.patch` on the HTTP library.
  - When AI generates a function that fetches and computes in one place, the refactor is the Listing 2 → Listing 3 move.

## Sandi Metz — Nothing is Something, RailsConf 2015 (https://www.youtube.com/watch?v=OMPfEXIlTVE)
**Core idea in one line: when code branches on a type or on nil, a missing concept is hiding in the gap. Name it, define its role, and inject an object to play it: null objects for "nothing", composition plus dependency injection instead of inheritance for variation.**

- **Mental model: message-centric, condition-averse OO**
  - In Smalltalk, which has only 6 keywords and no `if`, even booleans are objects you send messages to. Her Ruby demo: monkey-patch `TrueClass#if_true` to yield and `FalseClass#if_true` to do nothing, then add `if_false`, and put `Object`/`NilClass` versions in for truthiness. Each returns `self` so calls can chain. The point is not to adopt this but to imagine OO without `if`.
  - An `if` on an object's type or truthiness is a *type check*: "if I know what kind you are, I supply the behaviour; otherwise I send you a message." That keeps a procedural mindset inside OO code.
  - Conditions reproduce. A fallback string like `'no animal'` written inline gets copied everywhere, and changing it means shotgun surgery.
  - Preferences: *"I would rather know the name of a class than duplicate behaviour everywhere."* The corollary is to know as few classes as possible.

- **Null Object pattern ("the active nothing", Bruce Anderson's term)**
  - The scenario: `Animal.find(id)` returns nil for unknown IDs, and mapping over a list and calling `.name` blows up.
  - If you don't need the nil, then nil really is nothing: `compact` it away.
  - If you need to send it messages, nil is *something*. Fixes like `animal ? animal.name : 'no animal'`, `animal && animal.name`, and `animal.try(:name)` all expand into the same type-checking conditional, duplicated at every call site.
  - Fix: create `MissingAnimal` with `name` returning `'no animal'`, and return it in place of nil. Now every element responds to the same API and the call-site conditionals disappear.
  - The one remaining conditional lives in a *wrapper around the untrusted external API* (e.g. `GuaranteedAnimal.find(id)`, which returns `Animal.find(id) || MissingAnimal.new`). Your code talks only to your own trustworthy API. One condition, in one place.

- **The bigger abstraction (the House That Jack Built kata)**
  - Base `House`: an array of 12 phrase fragments, plus `phrase(n)` (`data.last(n).join(' ')`), `line(n)` ("This is ... ."), and `recite`.
  - New request: RandomHouse, which shuffles the fragments once. A subclass overriding `data` with a cached `shuffle` works in two minutes.
  - Next: EchoHouse, which duplicates every fragment. First a small refactor extracts `parts(n)` out of `phrase` so there's one place to change. Then a subclass overrides `parts` as `super.zip(super).flatten`. That works in three minutes too.
  - Then: RandomEchoHouse. Inheritance is stuck. You can inherit one side and duplicate the other, or copy everything. Modules or mixins are "just another form of inheritance", and multiple inheritance isn't the answer either. Many teams end up copying the whole class, which is at least honest.
  - **Diagnosis:**
    - Inheritance is for *specialisation*, not for sharing code.
    - The "is-a" test fooled us because of the names: "Random House is a house" sounds true.
    - A subclass that specialises a small part of its parent actually carries all the parent's surface area, so subclasses can't be combined.
  - **Technique: make things more alike to reveal how they differ.**
    - Move the array into a constant `DATA`, have `data` return it, and replace `super` with `data`. Now the two `data` methods differ only by `.shuffle`.
    - Lay the variants out like spreadsheet rows (Avdi Grimm's suggestion) and name the column. The column is **Order**: random is one ordering and the *original order is also an algorithm*, not nothing.
    - Ask whether "order" is a house. Clearly not. It's a *role*.
  - **Composition plus dependency injection:**
    - Define the role API `order(data)` with players `RandomOrder` (shuffle) and `DefaultOrder` (identity).
    - Inject it: `House.new(orderer: DefaultOrder.new)`, so House asks the orderer once to order its data.
    - Do the same for **Format**: `format(parts)` with players `DefaultFormatter` (identity) and `EchoFormatter` (zip with itself, then flatten). Intervene at the narrowest point.
    - Result: two roles times two players gives four behaviours (house, random, echo, random-echo) with *less* code than the inheritance version and no duplication.
  - The cost: at first composition gives you *more* code for the same behaviour. That's why people complain about OO. The payoff comes with the second variant.

- **Rules of thumb**
  - If you're talking to nil, use a null object. Stop scattering nil checks.
  - Beware inheritance, especially when you're specialising a small fraction of a big class. Use it to start if you like, but be ready to switch to composition.
  - There is never just one specialisation. A variant means there are at least two, and the "default" is a player too.
  - Recipe: isolate what varies, name the concept, define the role (API), inject the players at runtime.
  - Abstractions such as zero, null objects and default orderings "reveal their presence by the absence of code". Look for them in the gaps between conditionals.

- **Relevance to Sam**
  - Supplier integrations are a textbook case of "Order/Format roles". Viator, Coras and RateHawk are *players* of roles like `Searcher`, `Pricer` and `Booker`. Inject the provider object into the booking flow instead of scattering `if provider == 'viator':` branches, and don't subclass a fat `BaseProvider` and override bits, because combinations (say, cached plus provider X, or a sandbox version) will break the same way RandomEchoHouse did.
  - In Python, a role is a `typing.Protocol` or simply a function parameter (pass `order=shuffle` or `order=identity`). Strategy-by-callable is the light version.
  - Null objects in data work: return a `MissingPlace`/`UNKNOWN_LOCATION` sentinel with safe defaults, or empty DataFrames and empty lists, instead of `None`, so downstream transforms don't need `if x is not None` everywhere. Wrap untrusted APIs that return `None`, missing keys or `null` fields in your own adapter so the one condition lives there. (Note: in pandas pipelines, explicit null handling via NA is often clearer than sentinel objects. Apply the idea at the object/API layer.)
  - When AI proposes a subclass to tweak one method, ask "what's the role here?" first.

## Martin Fowler — Workflows of Refactoring, OOP 2014 (https://www.youtube.com/watch?v=vqEg37e4Mkw)
**Core idea in one line: refactoring isn't a separate project. It's woven into daily work through several triggers (TDD, litter-pickup, comprehension, preparatory, long-term), always in small behaviour-preserving steps, and justified purely by economics: a clean codebase lets you deliver faster.**

- **Mental model: the two hats** (Kent Beck)
  - *Adding function* changes behaviour. *Refactoring* preserves external behaviour and improves internal structure, normally on top of a test suite you run often.
  - Wear one hat at a time, but swap often. If you find a bug while refactoring, don't fix it under that hat. Switch hats, fix it, switch back. Always know which mode you're in.
  - Other hats: performance tuning (like refactoring, but aimed at speed rather than clarity) and sketching or exploration (throwaway code).
  - The refactoring rhythm: tiny steps, run tests after each, and if anything breaks, *undo* and retry in smaller steps. That's a different tempo from feature work.

- **The workflows**
  1. **TDD refactoring:** red (failing test), then green (simplest thing that works, ignoring how it fits the system), then refactor (remove duplication, reconsider how it fits). It's separated because beginners skip step three, and because writing and cleaning are different modes.
  2. **Litter-pickup refactoring** (the Boy Scout rule): you see yucky code, even your own and even code you're only *using*, and you improve it a bit. Not necessarily fully. Code you hit often converges to clean through many small improvements. Rarely touched code just gets its worst parts fixed. This is the conscious counterforce to design entropy, and it replaces the old "it works, don't touch it".
  3. **Comprehension refactoring:** after you finally puzzle out what code does, don't move on. Ward Cunningham: that understanding is fragile and lives only in your head. Move it *into the code* (renames, extractions) so the next reader doesn't have to repeat the detective work. "A puzzle is good in a detective novel, bad in code."
     - The reaction is the same for litter and comprehension. If the fix is quick (an IDE rename), do it now. If you're mid-feature with red tests, first get back to green (e.g. disable the in-progress test), refactor, then re-enable. If it's bigger, write a note (index card, scratch buffer) and do it right after the feature.
  4. **Preparatory refactoring:** "this feature would be easy if only the code were shaped differently." First refactor into that shape (hat on, tests green), then add the feature. This often takes *less* total time than forcing the feature into a bad fit. It's the payoff of evolutionary design, and it replaces heavy up-front design meant to foresee everything.
  5. **Planned refactoring:** cleanup as a line item in the project plan. It's a necessary evil and a *smell* that the other workflows aren't being done enough. It always needs justifying, which is why to avoid it. A perfect team never needs it because refactoring disappears into feature work.
  6. **Long-term refactoring:** for big structural problems like tangled module dependencies or an inherited codebase. Spend an hour or two agreeing on a target vision. Then, every time anyone works in that area, nudge it a little closer. Over weeks or months the dependencies break one by one until the last step is short. This keeps the code always releasable (continuous delivery), lets you learn along the way, and lets the path become clear once some mess is cleared.

- **Rules of thumb**
  - The essence of refactoring is *small steps*. Even large changes can be broken into tiny ones. If you can't see how, push in the direction of the goal and the path often appears.
  - Make refactoring an integral part of regular work. As soon as it becomes a separate task, it becomes a negotiation.
  - Decide how much to refactor by asking whether it will help me go faster: weigh how often the code is touched against the cost.
  - Code nobody touches, that works, can stay messy. Code that slows you down repeatedly is where to invest.

- **Why: the design stamina hypothesis**
  - Without design attention, feature delivery slows until "the code is fighting you". With good design, new features come from recombining good modules. The curves cross. Refactoring is how you stay on the good curve, since great up-front design almost never works.
  - Don't argue for refactoring as professionalism, quality or "clean code". As soon as you do, you've lost the argument with management. Argue *economics*: more functionality, delivered faster.
  - His controversial stance: don't ask management. Keeping the codebase healthy is the developer's professional responsibility, because only people in the code can see whether it slows them down. Not doing it is "stealing from your customer".

- **Relevance to Sam**
  - As a solo builder with AI help, he is the only one who can see the slowdown, so the economic test is his to apply. Refactor the modules he touches every week (supplier clients, the matching core), and leave one-off scripts that never get rerun.
  - The two hats map well onto AI-assisted work. Ask the AI for *either* a behaviour change *or* a pure refactor in one step, never both, and run tests between. Mixed diffs are where AI regressions hide.
  - Preparatory refactoring before adding a new supplier: first reshape the existing code so "a provider" is a clear role or module, then add the new provider as a small, separate change.
  - Comprehension refactoring is valuable after debugging an AI-written pipeline: put what you learned into names and extracted functions before moving on.
  - For legacy pipelines like the large locations script: set a target architecture once (pure core plus I/O shell, typed records), then move closer every time you touch it instead of planning a rewrite.

## Dave Farley — When Test Driven Development Goes Wrong, GOTO 2021 (https://www.youtube.com/watch?v=-4Ybn0Cz2oU)
**Core idea in one line: TDD's main value is design feedback. A test that is hard to write is telling you the design is wrong, and five common test anti-patterns (mostly from test-after and coverage targets) are signposts to better modularity, not reasons to abandon testing.**

- **Mental model**
  - Test-first creates a fast feedback loop on design quality: modularity, cohesion, separation of concerns, coupling. "If the test is hard to write, that isn't a problem with the test."
  - Most failures blamed on TDD are really failures of *after-the-fact unit testing*, which is a different practice. Tests written after the code couple to the implementation, so projects gradually "grind to a halt": every code change breaks tests, and every test change needs code changes.
  - Test coverage is a poor *target*. Good TDD teams get high coverage as a side effect, but mandating it produces bad behaviour. His example: an org paid bonuses for 80% coverage, hit it, and the tests asserted nothing.
  - Source list: James Carr's TDD anti-patterns blog post.

- **The five anti-patterns (symptom, cause, fix)**
  1. **The Liar:** passes, but asserts nothing useful.
     - Cause: coverage-chasing and test-after.
     - Harm: false sense of safety, and pure waste.
     - Fix: delete it or add real assertions. Test-first makes this nearly impossible, because you must see the test fail and should *predict how* it will fail.
  2. **Excessive Setup:** lots of work just to get the code into a testable state.
     - Cause: code written without testability in mind, and poor separation of concerns.
     - Harm: tight test–code coupling, fragile tests, and tests that check implementation rather than behaviour.
     - Example: a real test started all of Jenkins and configured an environment just to assert that a URL didn't contain "localhost". Test-first, that is a plain unit test of a string-building function.
     - Fix: better abstractions; carve the system into small modules ("one class one thing, one method one thing"). Nobody sane writes a heavy-setup test *before* the code, so test-first pushes toward simpler code.
  3. **The Giant:** many lines and many assertions (a real example had 9).
     - Cause: test-after, or tests grown organically as "component tests".
     - Harm: when assertion 7 fails, you can't tell whether it's the real failure or a side effect of something earlier. Unclear intent, poor documentation of behaviour, fragile.
     - Fix: split into focused cases. Prefer **one assertion per test** so a failure says exactly what broke. (Note: many practitioners read this as one *logical* assertion or behaviour per test. Several asserts about a single outcome are fine.)
  4. **The Mockery:** so many mocks or stubs that the system under test isn't tested at all.
     - Example: a test of Car and Engine where both were mocks, so only two interfaces with no implementation were exercised.
     - Cause: excessive setup, bad coupling, testing by rote because someone mandated it.
     - Fix: review the design, improve abstraction and separation, write tests first, and focus on desired *behaviour*, not technical implementation.
  5. **The Inspector:** breaks encapsulation to make assertions.
     - Example: a getter that existed in production code *only* so a test could call it.
     - Cause: test-after, 100% coverage goals, missing or improper dependency injection.
     - Fix: never compromise encapsulation for tests. Design for testability from the start. Use mocks or stubs at well-defined boundaries as *measurement points* rather than peeking at private state. Assert behaviour, not representation.

- **Decisions & trade-offs**
  - Mocks are legitimate as observation points at real boundaries. They are an anti-pattern when they replace the thing you're meant to test.
  - Unit test small behaviours. Starting the whole app to check a string is waste in cost, runtime and maintenance.

- **Checklist**
  - Did I see this test fail first, the way I predicted?
  - Is the setup small? If not, what responsibility should be split off?
  - Does the test name or assert one behaviour?
  - Am I mocking the thing under test?
  - Did I add any production API only for the test?
  - Is a coverage number driving this test?

- **Relevance to Sam**
  - AI-generated test suites often contain liars (asserting `result is not None`), giants, and mockeries (patching every function inside the module under test). Audit them with the checklist above.
  - Excessive setup around a pipeline step (needing a real DB, a full GeoNames file, or live supplier credentials) is a design signal. Extract the pure transformation so it can be tested on a five-row literal. This is the same move Bernhardt and Rhodes describe.
  - Don't add `_debug_internal_state()` accessors to a matcher just to test it. Assert on its outputs, such as matched IDs and scores.
  - Don't set coverage as a goal for his own projects. Use failing-first tests on the decision-heavy core.

