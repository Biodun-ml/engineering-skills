# fe-1 — cross-cutting principles

1. **Use the native element first; everything else is a patch.** `<button>`, `<a href>`, `<label>`, `<nav>/<main>`, `<ul>` give the accessibility tree its roles and names and give keyboard support for free. A styled `div` needs `role`, `tabindex`, key handlers and still falls short. First rule of ARIA: don't use ARIA when HTML can do it. (Frederik/freeCodeCamp a11y)
2. **The browser builds trees from text, then paints**: bytes → characters (charset) → tokens → nodes → DOM; the same for CSS → CSSOM; DOM + CSSOM → render tree → layout (geometry) → paint. Nothing is visible until paint. The accessibility tree is a parallel output from the same markup. (Hitesh Choudhary; Frederik)
3. **Classic scripts block parsing, and pending CSS blocks scripts.** Put CSS in head and keep it lean. Load scripts with `defer` (ordered, after parse) or `async` (independent, e.g. third-party widgets). Sending real HTML first and JS later (server rendering + hydration) paints sooner than a JS-only SPA. (Hitesh Choudhary)
4. **One thread, one call stack**: JS runs one thing at a time; anything slow on the stack freezes rendering and input. "Don't block the event loop" means no long synchronous work on the main thread: chunk it, move it to a worker, or move it to the server. (Philip Roberts; Lydia Hallie)
5. **Async is the host's job, not the language's**: `setTimeout`, `fetch`, DOM events and geolocation are Web APIs (C++ APIs in Node). The JS call only registers a callback and returns; the host does the work and queues the continuation. (Roberts; Hallie; Hitesh)
6. **Event loop ordering**: sync code → drain *all* microtasks (promise handlers, code after `await`, `queueMicrotask`, MutationObserver) → one task (timers, events, callback APIs) → microtasks again; rendering gets a turn only when the stack is clear (~every 16.6 ms at 60 fps). Recursive microtasks starve tasks and rendering. To yield to paint, schedule a task, not a microtask. (Hallie; Roberts)
7. **Timer delays are minimums**: the delay is time until *queued*, not until run; `setTimeout(fn,0)` means "after the current stack clears". High-frequency events (scroll/resize/input) flood the queue, so debounce/throttle them. (Roberts; Hallie)
8. **Flexbox is content-out and one-dimensional**: items start at max-content, shrink (default `flex-shrink:1`) toward min-content, grow only if `flex-grow>0`; grow/shrink are ratios, so plain `display:flex` gives unequal columns. `flex:1` (basis 0) or `width:100%` gives equal ones. Each wrapped row is independent; need aligned rows and columns → Grid. (Kevin Powell)
9. **Alignment needs free space**: `justify-content` (main axis) only distributes *leftover* space, so it does nothing if items are shrinking, something grows, or a column container has no height (use `min-height`). `align-items` (cross axis, default stretch) positions each item; `align-self` overrides one. Use `gap`, not margins. (Kevin Powell)
10. **TypeScript types are erased at runtime**: they catch bugs in the editor and CI only. External data (APIs, JSON, user input) still needs runtime validation; `as Type` on a fetch result is an unchecked promise. (Matt Pocock)
11. **TS is only as useful as its strictness**: `strict: true` (+ `noUncheckedIndexedAccess`), type-check in CI (`tsc` emits JS even with errors by default), and treat `any`, `as any`, `as unknown as` as code smells to justify. (Matt Pocock)
12. **Model branches as unions and narrow them**: literal unions (`"create" | "update"`) plus `typeof`/discriminant checks make each case explicit, autocomplete-able and compiler-checked; prefer inference over annotation everywhere else. (Matt Pocock)
13. **Never carry meaning by colour alone, and meet contrast**: ≥4.5:1 for body text (AA), 3:1 large, 7:1 for AAA; check placeholders and footers; pair status colours with icon + text. (Frederik)
14. **Names must stand alone**: link and button text has to make sense out of context ("Notify me", "About Skynet", not "click here"/"✓"); every input has a `<label for>`; placeholder is an example, never the label; group radios in `fieldset/legend`. (Frederik)
15. **Structure is navigation for assistive tech**: one h1, no skipped heading levels (headings are the top way screen-reader users navigate), landmarks, real lists; font sizes in `rem` so user font settings work. (Frederik)
16. **Announce dynamic changes and move focus deliberately**: `aria-live="polite"` for status/result updates (`assertive` only for urgent ones); after submit/modal open, `.focus()` the logical next element; add a skip link. Overuse of either disorients users. (Frederik)
17. **Verify with tools *and* by hand**: Lighthouse/axe/DevTools accessibility tree and contrast tooltip, plus keyboard-only and screen-reader passes; the DevTools Performance view and an understanding of the event loop for jank. Automated scores miss things. (Frederik; Roberts)
18. **Third-party widget rule (applies to Sam's Drive Button)**: load async/defer, no `document.write`, no heavy sync work on the host's main thread, scoped CSS, real `<button>/<a>` with descriptive text, no h1 hijacking, own background for guaranteed contrast. (Synthesis of Hitesh, Roberts, Hallie, Frederik, Powell)

---

## Frederik (freeCodeCamp / Scrimba) — Learn Accessibility - Full a11y Tutorial (https://www.youtube.com/watch?v=e2nkq3h1P68)
**Accessibility is mostly free if you use the right HTML element for the job; the browser turns correct markup into an accessibility tree that assistive tech reads, and everything else (ARIA, focus management) is a patch for when you could not.**

**Mental models / first principles**
- The browser builds two outputs from your markup: the visual UI and the **accessibility tree**, a simplified text-form model of the page (roles, names, states of headings, links, form controls). Screen readers, voice control, magnifiers and switch devices consume the tree, not your pixels. Two buttons styled identically look the same visually, but a `<div>` "button" shows up in the tree with no role, no name and no focusability.
- Semantic HTML = the correct element for the purpose. It feeds the tree automatically and gives keyboard support for free (`<button>`, `<a href>`, form controls are focusable with Tab and operable with Enter/Space without any JS).
- Accessible features help everyone: alt text shows when images fail on slow networks, clear link text helps skimmers, rem units respect user font settings, semantic tags help SEO (search engines weight keywords in headings/links more than in anonymous divs).
- Teacher cites a survey that ~96.5% of home pages have detectable a11y failures, most of them fundamentals (contrast, alt, labels, link text).

**Mechanisms**
- Inspect the tree: Chrome DevTools > Accessibility pane > "Enable full-page accessibility tree" (reload), then toggle the tree view. Firefox has an equivalent inspector.
- Contrast check: Ctrl/Cmd+Shift+C, hover text, read the contrast ratio in the tooltip. Ratios run from 1:1 (same colour) to 21:1 (black on white) (note: teacher said "1:21"; the convention is 21:1).
- WCAG contrast thresholds: AA = 4.5:1 normal text, 3:1 large text; AAA = 7:1 normal (4.5:1 large). Some backgrounds (saturated reds/greens) cannot reach 7:1 with any text colour, so the fix is to change the background or enlarge the text. (note: the teacher says level "A means not accessible" — that is wrong. A is the minimum conformance level, AA is the usual legal/industry target, AAA is aspirational and not required site-wide.)
- Text over images: either move text off the busy area (e.g. `display:block` on a word so it sits on its own line) or lighten the image with `background-color: rgba(255,255,255,.85); background-blend-mode: lighten;`.
- **Alt text**: every `<img>` gets an `alt` attribute. Informative images get a concise description (don't start with "image of"; the reader already says "image"). Include any text baked into the image if it carries meaning. Decorative images (backgrounds, flourishes, icons next to a text label like a search magnifier) get `alt=""` — an empty alt tells readers to skip; a *missing* alt may make the reader announce the filename ("background dot jpg"). Ending alt with a period gives a small pause.
- **Links**: must be `<a>`, recognisable as links by more than colour (underline is the default and best-understood cue), and the text must make sense out of context — screen reader users often pull up a list of all links. Not "click here"/"more"; wrap the meaningful phrase instead.
- **Card links**: wrapping a whole card in `<a>` makes the reader announce the entire card's text as one long link and loses nested structure (images, headings). Better: a real link inside the card on the heading/CTA, then stretch its hit area with CSS (`position:relative` on card, `::after` on the link with `position:absolute; inset:0`). Add hover cursor and make the CTA text say what the whole card does. `aria-label` on a link whose visible text is not descriptive.
- **Forms**: every input needs a `<label for="id">` matching the input's `id` (clicking the label focuses the field). Placeholder is NOT a label (inconsistent screen-reader support, disappears on typing); use it only as an example value ("e.g. Sherlock Holmes"), never a copy of the label. `<select>` and `<textarea>` are labelled the same way. Group radio buttons in `<fieldset>` + `<legend>` so readers announce the group and its question. Button text is its accessible name — "Notify me", not "✓" or "Done". Keep placeholder text above contrast thresholds too.
- **Landmarks**: `<nav>`, `<header>`, `<main>`, `<section>`, `<footer>` let screen-reader users jump between regions and hear "navigation", "main" on entry.
- **Lists**: consecutive related items (nav links, stats, footer links) belong in `<ul>/<ol>` + `<li>`; readers announce list start/end and item count. Remove bullets with `list-style-type:none` (note: Safari/VoiceOver drops list semantics when list-style is none; adding `role="list"` restores it — not covered in the video).
- **Font size in rem**: px does not follow the user's browser default font-size setting (it does follow zoom). 1rem = root font size (default 16px), so 20px → 1.25rem. A user who sets 32px default gets doubled text automatically.
- **Headings**: the most common way screen-reader users navigate. Rules: one `<h1>` per page, don't skip levels (h2 → h4 is wrong), choose level by structure, never by look; style in CSS.
- **ARIA** (Accessible Rich Internet Applications): attributes that add role/name/state where native HTML falls short. First rule of ARIA: don't use ARIA if a native element exists. A `div` button needs `role="button"`, `tabindex="0"`, plus JS for Enter/Space — and still lacks some native behaviour. Legit uses: custom toggles, sliders, tabs. A dark-mode toggle needs `role="switch"`, `aria-checked="true|false"` (updated by JS), `tabindex="0"`.
- **Live regions**: `aria-live="polite"` on a container whose content changes (form-submit status, filter result counts, notifications) makes readers announce the update at the next pause; `assertive` interrupts and is for urgent things only; `off` is default. `aria-controls` links the trigger to the region it updates.
- **Programmatic focus management**: after modal open, menu open, or form submit, move focus with `element.focus()` to the logical next place (e.g. the success message or a home link). Overuse disorients users.
- **Skip link**: first focusable element, `<a href="#main">Skip to main content</a>`, visually hidden off-screen until `:focus`, then slides in. Hide with a technique that keeps it focusable (off-screen positioning/transform), not `display:none`/`visibility:hidden`. Teacher adds `transition` (1s in, 3s out) so fast tabbers notice it.

**Testing**
- Manual: use a real screen reader on your page; tab through every interactive element.
- Automated: Lighthouse (0–100 a11y score with itemised issues), Firefox Accessibility Inspector, axe. Teacher also pastes code into an LLM and asks about contrast, missing labels and wrong elements.

**Rules of thumb / review checklist (for AI-written UI)**
- Any `div`/`span` with an `onClick` → should it be `<button>` or `<a>`? (link = navigates, button = does an action).
- Every `img` has `alt`; decorative = `alt=""`.
- Every input has an associated `<label>`; placeholders are examples.
- Link and button text understandable out of context; links distinguishable without colour.
- Contrast ≥ 4.5:1 body text (check placeholder and footer text — common failures).
- Don't convey state by colour alone (online/offline dot → add icon + text; ~300M people have some colour-vision deficiency).
- One h1, no skipped levels; landmarks present; lists for repeated items.
- Font sizes in rem; dynamic status messages in a polite live region; focus moved sensibly after actions.

**Relevance to Sam**
- Drive Button widget injected into third-party blogs: render it as a real `<button>`/`<a>` with descriptive text ("Book this trip to Lisbon", not "Go"), keep it keyboard-reachable, and don't hijack the host page's heading hierarchy (use no h1 inside the widget). Use rem so it respects host/user font settings, and check contrast against unknown host backgrounds (ship your own background).
- Tereka/Riselist forms: labels + fieldsets for radio groups, polite live region for "Saved" / validation messages, focus to the first error on submit.
- AI codegen often emits div-buttons and placeholder-only inputs — scan for those first; run Lighthouse/axe in CI.

## Kevin Powell — Learn flexbox the easy way (https://www.youtube.com/watch?v=u044iM9xsWU)
**Flexbox starts from each item's content-based "ideal" size, then grows or shrinks items to resolve leftover or missing space along one axis; almost every surprise comes from forgetting that starting point or that there is no leftover space to distribute.**

**Mental models / first principles**
- `display:flex` on a parent turns only its *direct children* into flex items. Their old display (block, inline, inline-block) stops mattering; only `display:none` still has effect. Flex items do not stack and stretch like blocks.
- Layout is content-out, not container-in. Each item's starting size (its flex basis when `auto`) is its **max-content** width: all text on one line, no wrapping. The floor is **min-content**: wrap at every opportunity, width of the longest word. Flexbox sizes items between those two intrinsic sizes.
- Parent sets the ground rules (direction, wrap, justify, align, gap); children have a lot of final control (grow, shrink, basis, align-self).
- Flexbox is **one-dimensional**: with `flex-wrap`, each row is laid out independently, so items on different rows don't line up into columns. That is the core difference from Grid (2-D, tracks shared across rows).

**Mechanisms**
- `flex-shrink` default **1**: items may shrink below max-content when the parent is too narrow, wrapping their text, down to min-content; after that they overflow. Setting it to 0 makes items overflow immediately. Default-on shrinking is "the magic" of flexbox.
- `flex-grow` default **0**: if set >0, leftover space on the main axis is shared among items. Grow and shrink are **ratios**, not sizes: an item with a larger ideal size stays larger after shrinking. That is why three cards with `display:flex` come out *unequal* widths — the card with the longest paragraph had the biggest ideal width.
- `flex-wrap: wrap` (default `nowrap`): items move to a new line instead of shrinking; each line grows independently.
- `gap`: spacing between flex items (now supported in all modern browsers); avoids margin hacks that must change per breakpoint.
- `flex-basis` default `auto` (= use width if set, else content size). Think of it as "width along the main axis". Differences from width: `flex-basis: 0` doesn't produce zero-width items — they stop at min-content — whereas `width: 0` really collapses them (leaving padding). Use basis or width, not both.
- **`flex: 1`** is shorthand for grow 1, shrink 1, **basis 0** (when basis is omitted from the shorthand, the spec sets it to 0, not auto). All items start from the same 0 basis, so leftover space is split equally → equal columns. Alternative: `width:100%` (or `flex-basis:100%`) on each child — they all want to be the parent's full width, so they are equal, and default shrink scales them down equally. Unequal padding between items breaks the equality because the algorithm works on content size.
- `flex-direction`: `row` (default) makes horizontal the **main axis**, vertical the **cross axis**; `column` swaps them. Kevin's responsive pattern: `display:flex; flex-direction:column; gap:…` by default, then switch to `row` in a `@media (min-width: …)` query. Gap keeps working both ways, so no margin juggling.
- `justify-content` = main axis, distributes **leftover** space: `flex-start` (default), `flex-end`, `center`, `space-between` (space only between items), `space-around` (half-size gaps at the edges; rarely useful), `space-evenly` (equal everywhere; the practical one).
- `align-items` = cross axis, positions each item individually within the line: default `stretch` (why flex columns are equal height), `flex-start`, `flex-end`, `center` (easy vertical centring). `align-self` on a child overrides the parent for that one item. There is no `justify-self` in flexbox (it exists in Grid, where items sit in cells). `align-content` only matters with wrapped multi-line flex containers — rare.

**Failure modes & gotchas**
- `justify-content` "does nothing" when there's no leftover space: items are shrinking (small screen) or some item has `flex-grow` and has already eaten the space.
- With `flex-direction: column`, `justify-content` acts vertically, but the parent's height is normally just the sum of its children, so there is nothing to distribute. Give the parent a height (prefer `min-height`, e.g. `min-height:100vh` for a full-screen mobile nav).
- Unequal columns from plain `display:flex` → use `flex:1` or `width:100%` on children.
- Overflow once items hit min-content (long words, URLs) → allow wrap, or handle long strings.
- Even experts mix up row/column and justify/align; with only two options each, try the other one.
- `flex-start`/`flex-end` are being joined by plain `start`/`end` (as in Grid); support was patchy at recording time (2021).

**Rules of thumb**
- Navigation bar: `display:flex; flex-wrap:wrap; gap` with no grow — items keep their natural size and wrap neatly.
- Equal-width card row: `flex:1` on children. Need cards to line up in both dimensions → use Grid instead.
- Use `gap`, not margins, for spacing between flex children.
- Select direct children with `.parent > *` to control flex items regardless of tag.

**Relevance to Sam**
- When reviewing AI-written CSS for the Drive Button widget or Tereka/Riselist layouts, the usual mistakes are: margins instead of gap, `justify-content` on a column container with no height, `width` and `flex-basis` both set, and expecting `display:flex` alone to give equal columns. A bookable-button row inside a blog article should be `flex-wrap: wrap` + `gap` so it degrades on narrow host columns rather than overflowing the host page.
## Hitesh Choudhary — How does a browser work? | Engineering side (https://www.youtube.com/watch?v=5rLFYtXHo9s)
**A browser is closer to an operating system than an app: it fetches bytes, turns HTML and CSS into object trees (DOM, CSSOM) in C++, combines them into a render tree, computes geometry, then paints — and JavaScript and CSS can block that pipeline.**

**Mental models / first principles**
- You never "run" HTML. HTML is text that the browser's rendering engine (written in C++) parses into in-memory objects; those objects, not your tags, are what get laid out and painted. Evidence: `document.getElementsByName(...)`/`querySelectorAll` hand you a NodeList of objects, never tag text. (note: the transcript at one point says HTML is "a parser on top of CSS" — he means C++.)
- Browser as near-OS, with parts: (1) **UI layer** (what you see/interact with); (2) **data/persistence** (cookies, localStorage, cached scripts, its own memory); (3) **browser engine**, split into the **rendering engine** (HTML/CSS → pixels) and the **JS engine** (e.g. V8); (4) **networking** stack (fetches every resource; even local files are treated as "loaded"); (5) **timers and other Web APIs** — `setTimeout`/`setInterval` are not part of the JS language; the browser provides them to JS (Node supplies its own implementations). This last point is the bridge to the event loop videos in this group.
- Two jobs of the browser: display data, and let the user interact with it. Both are hard.
- Engine implementations differ (Blink, WebKit, Gecko), but the core pipeline is the same across them, otherwise the web would break.

**Mechanism: the critical rendering path**
1. **Bytes**: the file arrives as raw bytes over the network.
2. **Characters**: bytes are decoded using the declared character encoding (`<meta charset="utf-8">`, or the HTTP header). Wrong/missing charset → garbled non-English text — declare it.
3. **Tokens**: the tokenizer recognises start tags, end tags, attributes, text (like a compiler picking keywords out of source).
4. **Nodes/objects**: each token becomes an object holding its tag name, attributes and content.
5. **DOM**: objects are linked into a tree with parent/child/sibling relations — the Document Object Model (literally: document → objects → model).
6. The same bytes → chars → tokens → nodes → tree pipeline runs for CSS, producing the **CSSOM**. DOM and CSSOM are built independently; the browser starts on CSS as soon as it meets a `<link rel=stylesheet>`.
7. **Render tree**: DOM + CSSOM are combined (only visible nodes, with their computed styles).
8. **Layout** (he calls it the math step): using viewport size, compute each box's position and size (px, %, etc.).
9. **Paint**: pixels drawn to screen. Nothing is visible until this point, however fast the earlier steps were. (note: modern engines add a compositing step after paint — layers combined on the GPU; not covered.)

**Blocking behaviour (the performance-relevant part)**
- A classic `<script>` (no attributes) is **parser-blocking**: when the parser meets it, DOM construction stops until the script is downloaded and run, because the script could change the DOM (e.g. `document.write`).
- Scripts also wait for pending **CSSOM**: a script might read computed styles, so the browser won't execute it until earlier stylesheets are parsed. So CSS is render-blocking and, indirectly, script-blocking — big CSS files delay everything.
- Fix: mark scripts `defer` or `async` so download happens in parallel without blocking parsing. (note: he blurs the two. `defer` = run after the HTML is parsed, in document order — the right default for app code; `async` = run as soon as downloaded, any order — for independent third-party scripts like analytics. Neither literally moves the tag; it changes when it runs.)
- Framework angle: a client-rendered React SPA ships a near-empty HTML shell and a big JS bundle, so the user sees nothing meaningful until the JS downloads and runs. The modern push (Remix, server rendering, then **hydration**) is to send HTML+CSS first so paint happens early, and send JS later to add interactivity. He says this ordering is still under debate.

**Rules of thumb**
- Declare charset first in `<head>`.
- CSS in `<head>` (needed to paint correctly), and keep critical CSS small.
- Scripts: `defer` by default; `async` for independent scripts; avoid plain blocking scripts in `<head>`.
- Prefer sending meaningful HTML from the server so first paint doesn't wait on JS.

**Gotchas**
- "Fast JS" doesn't help if the page can't paint because of render-blocking CSS or parser-blocking scripts.
- DOM access from JS deals with live objects, not the source HTML; what's in DevTools Elements is the DOM after scripts ran, not the original HTML.

**Relevance to Sam**
- Drive Button widget is a third-party script on someone else's blog. Load it `async` (or `defer`) so it never blocks the host page's parsing/painting (and so the bloggers' Core Web Vitals don't suffer from your widget). Don't use `document.write`. Inject CSS scoped to your widget (or Shadow DOM) so you don't trigger large style recalculation or clash with the host CSSOM.
- For Tereka/Riselist: server-rendered HTML gives earlier first paint than a JS-only SPA, a direct input to choosing between server HTML (+htmx) and a client-heavy React app.
## Matt Pocock — TypeScript Speedrun: Crash Course for Beginners (https://www.youtube.com/watch?v=YmxwicpROps)
**TypeScript is JavaScript plus a compile-time type layer that is erased before anything runs; its value is catching mistakes in the editor and CI (with `strict` on), not safety at runtime.**

**Mental models / first principles**
- TS = all of JS + a type system. Types exist only at development/compile time: `tsc` strips interfaces and annotations and emits plain JS. At runtime, "JavaScript is still JavaScript" — TS adds nothing to the shipped code and cannot protect you from bad data arriving at runtime (API responses, `JSON.parse`, user input). Those need runtime validation separately (e.g. a schema validator; not covered in the video).
- **Inference first**: you don't annotate everything. `const s = "hello"` is inferred; callbacks like `names.map(name => …)` get their parameter types from context (contextual typing), and changes propagate through chains of `.map`.
- TS flags two kinds of problems: (1) things that would throw at runtime (calling a non-function: "Type 'User' has no call signatures" = "this is an object, not a function"); (2) **non-exception failures** — legal JS that's probably a bug, e.g. reading `user.location` when no such property exists (JS just gives `undefined`), typos like `toLocaleLowercase`, functions referenced but not called.
- Tooling payoff: types (including `@types/*` for JS libraries like Express) give autocomplete, go-to-definition (Cmd/Ctrl-click) and discoverability without reading docs.

**Mechanisms**
- Install: usually as a local dev dependency (`npm i -D typescript`); global is fine too. `tsc file.ts` emits `file.js`.
- **By default `tsc` still emits JS even when there are type errors** (it reports them but compiles). So the real enforcement is a CI step that type-checks every commit (`tsc --noEmit` in a GitHub Action is the common setup). That makes PR review cheaper because the basics are guaranteed.
- `tsconfig.json` at project root, next to package.json. **`"strict": true`** turns on the recommended rule set, including `noImplicitAny` (un-annotated parameters become errors instead of silently `any`) and `strictNullChecks` (`null`/`undefined` must be handled). Pocock also recommends **`noUncheckedIndexedAccess`** (indexing an array/record returns `T | undefined`). Turning strict off is only defensible for incremental migration of a huge JS codebase.
- Everyday types: `string`, `number`, `boolean`; arrays `string[]` or `Array<string>` (generic form — think "type function" taking a type argument). `[number]` is a **tuple** (fixed length/positions), not an array.
- `any`: escape hatch that switches checking and autocomplete off. Exists because JS is untyped at heart; beginners should avoid it, and many teams ban it with ESLint. Treat it as a code smell.
- Annotations: `const u: User = {...}` checks the object; parameter and return annotations on functions (a declared return type that is neither `void` nor `any` must actually return).
- Object types inline `{ x: number; y: number }` or named via `interface Coords {...}` or `type Coords = {...}` — interchangeable for plain objects. Optional property `location?: string` reads as `string | undefined`.
- **Union types** `string | number`, plus **narrowing**: inside `if (typeof id === "string")` TS knows it's a string (string methods available); in the else branch it's a number. Unions model the branches of your program (e.g. one of several actions).
- `type` can alias anything (unions, functions, objects); `interface` is for object shapes and supports `extends`. Types combine with intersections `A & { honey: boolean }`.
- **Type assertions** `x as HTMLCanvasElement` — "I know better than the compiler". You can force an incomplete object into a type, so it is unsafe. TS refuses wildly unrelated conversions (`"Matt" as number`) unless you go via `as unknown as number`. `as any` is the classic escape — avoid.
- **Literal types**: `const name = "Matt"` infers the literal `"Matt"`; `let` infers `string` (can be reassigned). Object properties stay widened (`string`) because objects are mutable, unless frozen/readonly. Unions of literals (`"create" | "update"`) act like lightweight enums and give autocomplete at call sites.
- `null`/`undefined` are types that go in unions (`string | null`); with strict on, you must handle them.

**Rules of thumb (for reviewing AI-written TS)**
- Check `tsconfig.json`: `strict: true` (+ `noUncheckedIndexedAccess`). If an AI turned strict off to "fix" errors, that's a red flag.
- Grep for `any`, `as any`, `as unknown as`, and non-null `!` — each one is a place the type checker was silenced; ask why.
- Type-check in CI; don't rely on the build failing (it won't by default).
- Prefer unions of literals for statuses/actions; narrow with `typeof`/discriminant checks rather than asserting.
- Types at the boundary (fetch/JSON) are promises you wrote, not facts: validate external data at runtime.

**Gotchas**
- Error messages are jargon-heavy ("no call signatures", "not assignable to"); "Type X is not assignable to type Y" = you put an X where a Y is required. Pocock's VS Code extension "Total TypeScript / TS Error Translator" rewrites them in plain English.
- The handbook leads with classes; you don't need classes to use TS.

**Relevance to Sam**
- Sam's supplier integrations (Viator, RateHawk) and booking flows are full of union-shaped data (booking status, provider variants). Literal unions + narrowing make every branch explicit and let the compiler flag unhandled cases. Since types are erased, provider JSON entering a Tereka/Drive Button frontend still needs runtime validation; `as Booking` on a fetch result is a lie the compiler can't catch.
## Philip Roberts (JSConf EU 2014) — What the heck is the event loop anyway? (https://www.youtube.com/watch?v=8aGhZQkoFbQ)
**The JS engine does exactly one thing at a time on one call stack; concurrency comes from the host (browser Web APIs or Node's C++ APIs) doing slow work elsewhere and handing callbacks back through a queue, which the event loop feeds to the stack only when the stack is empty.**

**Mental models / first principles**
- The engine (V8) is just a **heap** (memory allocation) and a **call stack**. Grep V8's source for `setTimeout`, DOM, or HTTP request and you find nothing: those are **Web APIs** provided by the browser around the engine. The full picture is: engine (heap + stack) + Web APIs + **callback (task) queue** + **event loop**. Node has the same shape with C++ APIs (libuv threads) in place of Web APIs.
- Single-threaded = one call stack = one piece of JS running at a time. "Blocking" has no formal definition: it simply means slow code sitting on the stack (a huge loop, a synchronous network request, heavy image processing).
- Why it matters in browsers: while the stack is busy, the browser cannot **render**, cannot respond to clicks, cannot even show text selection. Hence async callbacks are the norm: almost no browser or Node API blocks.

**Mechanisms**
- **Call stack**: calling a function pushes a frame; returning pops it. The script itself is the bottom "main" frame. An error's stack trace is a snapshot of the stack at that moment (`foo ← bar ← baz ← anonymous`). Unbounded recursion overflows ("Maximum call stack size exceeded" — Chrome killed it after ~16,000 frames in his demo).
- **`setTimeout(cb, 5000)`**: the call hands `cb` and the delay to the browser's timer and returns at once (its frame pops). The engine carries on. When the timer fires, the Web API cannot inject `cb` into running code — it pushes `cb` onto the **task queue**.
- **Event loop's one job**: look at the stack and the queue; if the stack is empty, take the first queued callback and push it onto the stack (which runs it).
- **`setTimeout(fn, 0)`** means "run fn after the current stack clears" — a way to defer work, not run it now. Output order of `log("hi"); setTimeout(()=>log("there"),0); log("JSConf")` is hi, JSConf, there.
- XHR/fetch, DOM events (click handlers) work the same way: browser does the work/listening, queues the callback, event loop runs it when the stack is free. 100 rapid clicks queue 100 callbacks, processed one by one.
- **Timeout delay is a minimum, not a guarantee**: four `setTimeout(…,1000)` calls queue four callbacks; later ones wait their turn after the delay expires.
- "Callback" means two things: any function passed to another function (e.g. `Array.prototype.forEach` runs it **synchronously** on the current stack), versus an **asynchronous** callback that goes through the queue later. Don't assume a callback API is async.
- **Rendering and the loop**: the browser wants to repaint every ~16.6 ms (60 fps). A render is like a high-priority queued job that also needs an empty stack; it gets priority over normal callbacks, but cannot interrupt running JS. A slow synchronous loop over an array blocks all renders until it finishes; splitting the work into queued async chunks (one per item) lets the browser render between chunks, so the UI stays responsive.
- **Flooding the queue**: high-frequency events (scroll fires roughly every frame) with slow handlers don't block the stack outright but pile up a backlog of callbacks. Fix: **debounce/throttle** — do the expensive work at most every N ms or after events stop.

**Rules of thumb**
- "Don't block the event loop" = never put long-running synchronous work on the main thread. Break it into chunks, or move it to a Web Worker (his visualiser tool, Loupe, itself runs code in a worker).
- Debounce/throttle scroll, resize, input handlers.
- Use `setTimeout(…,0)` (or modern equivalents) only to yield/defer deliberately, knowing it runs "next-ish", not at a fixed time.
- Treat timer delays as lower bounds.

**Gotchas**
- Synchronous XHR exists in browsers and freezes the whole page; never use it.
- A busy main thread is invisible in code review but obvious to users: frozen buttons, laggy scroll. (In today's terms this is what the INP Core Web Vital measures — note: INP postdates the talk.)
- (note: this 2014 talk predates widespread Promises; it describes one task queue. Promise callbacks use a separate, higher-priority microtask queue — see Lydia Hallie's video below.)

**Relevance to Sam**
- Same mental model as Node backends: one JS thread plus a pool doing I/O. For a Drive Button widget running on a blog's main thread, any heavy synchronous work (e.g. scanning a long article's text for place names in-browser) blocks the host page's rendering and scrolling. Do such work server-side, in chunks, or in a worker. For Tereka (client-side money calculations over many rows) chunk or offload big recomputations, and debounce input-driven recalculation.
## Lydia Hallie — JavaScript Visualized: Event Loop, Web APIs, (Micro)task Queue (https://www.youtube.com/watch?v=eiC58R16hb8)
**There are two queues, not one: callback-based Web APIs feed the task queue, promises/await feed the microtask queue, and the event loop drains the *entire* microtask queue every time the stack empties, before taking one task.**

**Mental models / first principles**
- Runtime parts: JS engine (call stack + memory heap), Web APIs, task queue, microtask queue, event loop. JS is single-threaded (one call stack), so a heavy computation freezes the whole program until it finishes.
- Web APIs are interfaces to browser features (DOM, `fetch`, `setTimeout`, geolocation, sensors, camera). Some let you **offload** long-running work: the JS call merely *initiates* the work and registers what should happen afterwards, then pops off the stack. The work itself runs in the browser in the background.
- Async Web APIs come in two styles: **callback-based** (setTimeout, geolocation `getCurrentPosition(success, error)`, event handlers) and **promise-based** (`fetch`). The style decides which queue the continuation lands in.

**Mechanisms**
- Each function call creates an execution context pushed on the stack; returns pop it (illustrated with nested `logThreeAndFour → logThree → console.log`).
- Callback-based flow (geolocation): call registers callbacks and starts the permission prompt in the background, pops immediately; the page stays responsive while the user decides; on "Allow" the success callback is pushed to the **task queue** (also called callback queue / macrotask queue) — never straight onto the stack, since that could interrupt running code. Event loop moves it to the stack when the stack is empty.
- **setTimeout's delay is the delay until the callback is *queued*, not until it runs.** If the stack is busy for seconds, the callback waits in the task queue.
- **Microtask queue** holds only: `.then/.catch/.finally` handlers, the remainder of an `async` function after an `await`, `queueMicrotask` callbacks, and `MutationObserver` callbacks.
- **Priority rule**: when the stack empties, the event loop runs microtasks until the microtask queue is completely empty, then takes **one** task from the task queue, then checks microtasks again, and so on.
- `fetch` walkthrough: `fetch()` creates a pending promise object (state pending, result undefined, no reactions yet) and starts the network request in the background; `.then(cb)` registers a promise reaction record; synchronous code continues ("end of script" logs first); when the response arrives the promise becomes fulfilled with a Response object and the reaction handler is pushed to the **microtask** queue, then run.
- If a promise is already resolved (`Promise.resolve().then(cb)`), the handler is queued as a microtask immediately — but still runs only after the current synchronous code.
- **Starvation**: a microtask that keeps scheduling microtasks keeps the event loop in the microtask phase forever; task-queue callbacks (and rendering) never get a turn and the page freezes. (She mentions Node has a guard on recursive `process.nextTick`; note: treat that detail as approximate.)
- **Promisify** a callback API for readability: `new Promise((resolve, reject) => navigator.geolocation.getCurrentPosition(resolve, reject))`.

**Quiz to internalise (answer 5 1 3 4 2)**
```js
Promise.resolve().then(() => console.log(1));
setTimeout(() => console.log(2), 10);
queueMicrotask(() => { console.log(3); queueMicrotask(() => console.log(4)); });
console.log(5);
```
Sync first (5); stack empty → drain microtasks (1, 3, and the newly added 4, because the queue must be empty before tasks); then the timer task (2).

**Rules of thumb**
- Ordering: synchronous code → all microtasks (promise continuations, code after `await`) → next task (timers, events, callback APIs) → microtasks again.
- `await` doesn't block the thread; it splits the function, and everything after it runs as a microtask later.
- Never build recursive microtask loops to "process work in chunks": that blocks rendering just like a sync loop. To yield to rendering/input, schedule a **task** (`setTimeout`, `MessageChannel`, `scheduler.yield` where available), not a microtask.
- Wrap old callback APIs in promises once, at the boundary, so the rest of the code can use async/await.

**Relevance to Sam**
- Explains real bugs in frontend code that AI writes: "state not updated yet" after `await`, timers firing later than expected, UI not repainting during a long `async` loop (each `await` on an already-resolved promise yields only to microtasks, not to paint). Same model applies to Node services for supplier calls: a long chain of resolved promises can starve I/O callbacks. For the Drive Button widget, fetch trip data with `fetch().then`/`await` and render on resolution, but keep any processing of the response short so the host page stays responsive.
