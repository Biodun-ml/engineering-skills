# fe-2 — cross-cutting principles

1. **Choose where the contract lives; errors surface there.** Annotate at the source (params, return types, object literals) so mistakes are flagged where they are made, not downstream. — Pocock, TypeScript Crash Course
2. **Types are erased at runtime.** TS gives zero protection against bad external data; `as`, `any` and `@ts-ignore` are lies to the checker, and `any` spreads. Validate supplier/API payloads at runtime at the boundary; run project-wide `tsc` in CI. — Pocock
3. **Model states as discriminated unions,** not bags of optional fields; prefer string-literal unions to enums. — Pocock
4. **Start minimal, add complexity only when the layout demands it:** `display:grid` → `gap` → columns with `fr`/`repeat` → spans → explicit lines/areas last. Let the implicit grid make rows. — Powell, Learn CSS Grid
5. **Flex vs grid is content-out vs layout-in.** Flex respects intrinsic size (nav, chips, button groups); grid lets the parent impose shared tracks (cards, page layout). Nest them freely; lots of child-level flex hacks signal you want grid. — Powell, Flexbox or grid
6. **Mobile-first, parent-controlled responsiveness:** base single column, add columns/spans in `min-width` queries; `grid-template-areas` lets you redefine layouts on the parent only. Keep DOM order meaningful because placement does not change reading/tab order. — Powell, CSS Grid
7. **Every web interaction is DNS → HTTP request (method, headers, body) → response.** Only the payload (HTML vs JSON) and who renders it differ. Debug from the Network tab first. — Schwarzmüller, How the Web Works
8. **Rendering location is the central architecture choice.** Server-rendered HTML (HTMX hypermedia, React Server Components) sends finished content in one round trip; client rendering adds HTML → JS → fetch → render steps, costing LCP and complexity. — Gross (HTMX); Cook (RSC); Delaney (performance)
9. **Default to server, push interactivity to small leaves.** In RSC, `"use client"` is a one-way door; keep it at the smallest interactive component and keep secrets/heavy libs in server-only code. — Cook, RSC
10. **Simplicity and closeness to the platform buy longevity.** Fewer dependencies = less churn; boring, stable tools (forms, links, Rails/Django, server templates) solve most CRUD apps. Don't rewrite working apps for fashion; trial new approaches on internal tools. — Gross, HTMX
11. **API design for longevity:** get the core concept right, never break backward compatibility (config flags for old behaviour), say no to features, use an event-driven extension mechanism, start a new project for a new idea. — Gross
12. **Locality of behaviour:** keep behaviour next to the element/template it affects (HTMX attributes, template fragments) so code is readable in one place. — Gross
13. **Measure Core Web Vitals per element:** LCP ≤ 2.5 s (compress, CDN, preload, `fetchpriority`, no render-blocking JS), interactivity (FID → now INP ≤ 200 ms: less main-thread JS, workers, lazy loading), CLS (dimensions/`aspect-ratio` on all media, no injected shifts). — Delaney, Web performance
14. **Use real tools, at scale:** Network waterfall, Lighthouse, Web Vitals extension (names the offending element), Unlighthouse across every page and in CI; complement lab data with field data. — Delaney
15. **Third-party scripts are a performance liability for the host page.** Any embedded widget must be async, small, non-blocking and reserve its space. — Delaney (applied to Sam's Drive Button)
16. **HTTPS everywhere; no secrets in client code.** Anything shipped to the browser is public; server-side (RSC, route handlers, server templates) is where credentials live. — Schwarzmüller; Cook

**Sam-specific takeaways:** (a) Drive Button = third-party embed → zero CLS, async load, flex-wrap buttons, CORS + HTTPS, server does the heavy lifting; (b) internal data/admin tools → Python server templates + HTMX; (c) Tereka/Trameter UIs → server-first rendering, strict TS, runtime-validated supplier data, grid for page layout.

---

## Matt Pocock — TypeScript Crash Course with Matt Pocock (VS Code live stream) (https://www.youtube.com/watch?v=p6dO9u0M7MQ)
**TypeScript is a compile-time contract checker layered on JavaScript: you decide *where* each contract lives, and that choice decides where errors show up.**

**Mental models / first principles**
- TypeScript is a strict reviewer looking over your shoulder (Pocock's "red-pen teacher"): it draws your eye to places where values could be the wrong shape. It adds nothing at runtime: types are erased when transpiled to JS, no runtime checks, zero bundle-size cost. So TS only protects you as far as you keep its contracts honest.
- Functions are the main "currency" of contracts. A parameter annotation (`a: number`) is a promise about how a function may be called; TS enforces it at every call site.
- Inference vs annotation: TS infers when it knows for certain (`const firstName = "Matt"` is a string). It refuses to guess parameter types from the body because a function can be called from anywhere, and `a + b` is valid for strings (concatenation) as well as numbers. It is deliberately conservative rather than "galaxy-brained".
- The location of an annotation decides where the error appears. Annotate a variable (`const defaultUser: User = {...}`) and the error moves to the object literal where the mistake is, not to some later function call. Add a return type (`(): number`) and errors happen inside the function body rather than leaking out to callers.

**Mechanisms**
- Primitives: `number` (no int/float distinction in JS), `string`, `boolean`. Wanting "integers only" needs advanced branded/opaque types plus a type predicate; not a beginner tool.
- `implicit any` errors come from `strict`/`noImplicitAny` in tsconfig. VS Code only checks the open file (and whatever it imports); run `tsc` (e.g. `npx tsc` / `yarn tsc`) to type-check the whole project — the "hand in all your homework" step. Use it in CI.
- Object shapes three ways: inline (`params: { first: number; second: number }`), a `type` alias, or an `interface`. Inline literals make error messages print the full structural shape, which gets huge; named types give readable errors ("not assignable to parameter of type Params").
- `type` can alias anything (string, arrays, unions, objects); `interface` describes only objects and supports `extends`/`implements`. Pocock: for plain objects the choice practically does not matter (he benchmarked performance claims); just be consistent.
- Optional properties `last?: string`; optional params `last?: string` or a default value (`last = "Pocock"`), but not both (TS errors). A required param cannot follow an optional one. Rest params `...otherNames: string[]` are typed as an array.
- Destructured params: runtime names on the left, type on the right: `({ first, second }: Args)`. You cannot annotate inside the destructuring pattern.
- Annotation (`: User`) vs cast (`as User`): annotation says "this slot must hold a User" and checks it; `as` asserts it is one and suppresses the check — lying to the teacher. Pocock compares `as` to CSS `!important`.
- `any` disables checking and *spreads*: `add(a: any, b: any)` makes the return `any` too, so typos like `toExponental` on the result stop being caught. `// @ts-ignore` silences the next line. Lint rules can warn on both.
- Definite assignment: a `let x: User;` used before assignment in the same scope errors; across function scopes TS does not track it, so `undefined` can still slip through at runtime.
- Literal types and unions: `role: "admin" | "user" | "super-admin"` restricts to exact strings and gives autocomplete; extract to `type Role = ...` and reuse.
- Intersection `A & B` merges object types. Combining a base type with a union of variants keyed on a shared field gives a **discriminated union**: `{role:"admin", adminPassword:string} | {role:"user"} | {role:"super-admin", superAdminPassword:string}`. Setting `role` narrows which other fields are required/allowed.
- Indexed access types derive types from types: `type Role = UserRoleAttributes["role"]` — the entry point to type-level programming.

**Decisions & trade-offs**
- Annotation `:` over `as` nearly always; reserve `as`/`any` for advanced cases where you truly know more than the compiler.
- Prefer string-literal unions over `enum`: enums emit runtime code and Pocock thinks TS would not add them today. Counter-argument from chat: enums centralise renames. (note: with `erasableSyntaxOnly` / Node type-stripping, enums are now further discouraged.)
- Shared types go in a common `types.ts`; types used in one place stay next to their use, like utility functions.

**Rules of thumb (for reviewing AI-written TS)**
- Turn on `strict`; run `tsc --noEmit` across the project, not just editor squiggles.
- Grep for `any`, `as`, `@ts-ignore`, `!` — each is a place where the contract is a lie. Ask why each one exists.
- Annotate at the boundary where data is created (object literals, function params, return types) so errors land at the source.
- Model states as discriminated unions (e.g. booking `status: "pending" | "confirmed" | "failed"` with per-status fields) instead of a bag of optional fields.
- Remember TS does not validate external data: API/supplier JSON typed with `as Booking` is unchecked. Validate at runtime at the edge (e.g. a schema validator) and derive the type from it.

**Failure modes**
- `any` leakage silently turning off checks downstream.
- Casting to satisfy the compiler, then crashing at runtime with `undefined`.
- Tests passing at runtime while types are wrong (or vice versa) — the two are independent checks.
- Relying on the editor (per-file) instead of a project-wide `tsc`.

**Relevance to Sam:** for the Drive Button widget and Tereka, the highest-leverage review checks are strict mode, no `as` on fetch responses, discriminated unions for UI/booking states, and runtime validation of supplier payloads (Viator/RateHawk shapes) before they enter typed code.

## Kevin Powell — Learn CSS Grid the easy way (https://www.youtube.com/watch?v=rg7Fvvl3taU)
**Start grid with the minimum (display, gap, columns), let the browser create rows, and only add explicit placement when the layout actually demands it.**

**Mental models / first principles**
- Grid looks complicated because of its many properties, but most layouts need four or five of them. Complexity should be added only when a specific layout problem forces it.
- Unlike flexbox, switching a container to `display: grid` usually changes almost nothing visually (single column), so it is a calm starting point. One subtle change: grid and flex children no longer have collapsing margins, so stacked paragraphs can suddenly look double-spaced.
- **Explicit vs implicit grid:** you declare columns (explicit); when items run out of columns, the browser auto-creates new rows (implicit), sized to the tallest item in the row (like flex items stretching). This matches the web's default flow, so let it work.
- The DOM order drives auto-placement: items fill the next empty cell in source order. Pinning one item to a specific line pushes the others into the next free cells; grid will not overlap items unless you force it.

**Mechanisms**
- `gap` for all spacing between tracks (both axes, never on the outer edges). Powell recommends it over margins "100% of the time" inside grids.
- `grid-template-columns`: px, %, or `fr` (a fraction of free space; `1fr 1fr 1fr` = equal columns). `repeat(4, 1fr)` avoids repetition.
- Spanning: `grid-column: span 2` — location-independent, so it can be a reusable utility class (`.grid-col-span-2`). He prefers long, self-explanatory class names over cryptic ones.
- Line-based placement: lines are numbered from 1 at the start edge. `grid-column: 4` sets only `grid-column-start`; `grid-column: 3 / 5` sets start/end; `grid-row: 1 / span 2` or `1 / 3` are equivalent. `grid-row-start: 1` lifts an item back to the top row.
- Shorthand gotcha: a shorthand like `grid-row` in a utility class can override a `grid-row-start` set elsewhere. When you need specific lines, set them on the component selector rather than via utilities.
- Design-reading technique: draw lines over the mockup (e.g. in Figma) to count real columns and rows before writing CSS; it is easy to miscount (he first wrote 3 columns when the design needed 4).
- DevTools grid overlay (Firefox, Chrome, Safari) shows line numbers and area names. Use it whenever touching line numbers.

**Responsive approach (mobile-first)**
- Base styles: `display: grid; gap` only (single column, items stacked in DOM order). In `@media (min-width: 50em)` add columns, spans and placements. Spans belong inside the media query because they only make sense once there are multiple columns.
- `repeat(auto-fit, minmax(...))` can avoid media queries for uniform card grids but does not fit irregular layouts like this one.

**Second approach: `grid-template-areas`**
- Each quoted string is one row; each word is a named cell; repeating a name across cells spans it: `"one one two five" "three four four five"`. Children get `grid-area: one` (no quotes there, unlike the template — a common slip). Put each row string on its own line for readability.
- Payoff: at each breakpoint you redefine only the parent's template; children never change. You can create several distinct layouts per breakpoint cheaply.
- Columns generated from areas are auto-sized by content and become uneven; fix once with `grid-auto-columns: 1fr` (same for `grid-auto-rows`).
- Use meaningful area names (e.g. `header`, `sidebar`, `price`) rather than numbers in real code.

**Decisions & trade-offs**
- `span N` (portable) vs line numbers (precise but brittle) — prefer span.
- Line-based placement is good for understanding; template areas cost more setup but scale better when layouts change a lot across breakpoints.
- Do not declare `grid-template-rows` unless you have a specific sizing need.

**Rules of thumb / checklist**
1. `display: grid` 2. `gap` 3. columns (`repeat(n, 1fr)`) 4. spans 5. explicit lines only if still needed 6. for multi-layout components, template areas + `grid-auto-columns: 1fr`.

**Failure modes**
- Forgetting `display: grid`.
- Visual order diverging from DOM order through placement — screen readers and keyboard tab order follow the DOM, not the grid (note: this a11y consequence is not stated in the video but follows directly from his "items follow DOM order" point; keep the source order meaningful).
- Overly long templates that break when an item is added or removed.

**Relevance to Sam:** when AI writes layout CSS for a Tereka dashboard or Trameter results page, check it is mobile-first, uses `gap` not margin hacks, uses `fr`/`repeat` rather than percentage math, and that any reordering via placement does not break reading/tab order.

## Carson Gross — HTMX in 2025 | PodRocket (LogRocket) (https://www.youtube.com/watch?v=kKAP6cUwOpo)
**Hypermedia (server returns HTML, any element can request and swap any part of the page) is a stable, simple alternative to JSON + client-side rendering — and stability/simplicity are features that compound over a system's lifetime.**

**Mental models / first principles**
- Two architectures: (a) **hypermedia exchanges** — the server sends HTML and the browser inserts it; (b) **JSON APIs + client templates/reactive frameworks** — the server sends data and client code builds the UI and owns state. HTMX (successor to Intercooler.js, 2013) generalises hypermedia controls so any element, on any event, can issue any HTTP method and place the returned HTML anywhere.
- In a hypermedia app most state is synchronised through HTTP requests returning new HTML, so client-side scripting can be a light touch: direct DOM manipulation (vanilla JS, jQuery, Alpine) rather than a reactive state system.
- **"100-year web service"** (Alex Petros' idea): the simpler the system and the more it leans on the platform (HTML, HTTP, browser built-ins), the longer it lasts. Each dependency adds churn: upgrades, cross-dependency conflicts, rewrites. Static-site generators and old Gatsby/React apps are cited as upgrade pain.
- **Stability as a feature:** Game Boy analogy — a worse screen won on battery life and simplicity. Developers chase the new partly from fear of becoming obsolete, which pulls attention away from boring tools that already work (Rails, Django). Most apps are reservations, inventory, forms — well-solved problems.
- Many developers who started on React do not know how a plain HTML form or link round-trip works. Gross recommends learning the original web model first (first chapters of the free book *Hypermedia Systems*: a links-and-forms app with no JS, in Python).

**Mechanisms**
- Core HTMX API (unchanged in spirit since Intercooler): HTML attributes on an element → on an event, issue a request → swap the HTML response into a target chosen by CSS selector. Reuse of existing concepts (CSS selectors) instead of inventing new ones kept it stable.
- **Template fragments:** mark a section of a server-side template so the server can render only that chunk for a partial update. Better than splitting into "include" files because it preserves **locality of behaviour** — you can read the whole page template in one place. Many template engines now support it.
- HTMX encourages putting behaviour on the element itself (locality) rather than in separate files.
- Progressive enhancement (site works with JS disabled) is possible with HTMX but takes deliberate effort; Unpoly is designed for it by default.
- The "Triptych" proposal to put HTMX ideas into HTML itself: (1) PUT/PATCH/DELETE from forms; (2) buttons as standalone hypermedia controls without a form; (3) **transclusion** — place a response into any element (today only possible for iframes via `target`). The old full-page-reload model loses scroll position and flashes; partial replacement fixes that.

**Library-design lessons (transferable to any API Sam builds)**
- Get the core concept right first; a decade of Intercooler validated it.
- Preserve backward compatibility: when behaviour must change (1.x → 2.x), add a config flag to restore the old behaviour instead of removing it. Costs some code cruft; buys trust.
- When the idea changes substantially, start a new project (HTMX vs Intercooler 3; "fixi", ~89 lines, as a minimalist variant) rather than breaking the old one.
- Say no to features; once added they cannot be removed. Use an **extension mechanism**, event-driven, so ideas can live outside core (WebSocket/SSE support moved from core to extensions in 2.0). Ask "does this serve the core idea?"

**Decisions & trade-offs**
- Do NOT rewrite a working React app in HTMX. Good entry points: neglected internal tools; or an app that is genuinely failing where the team understands traditional web apps.
- HTMX intentionally ships no components or client-scripting story; pair with DaisyUI, Alpine, Surreal, vanilla JS. Freedom vs lack of an ecosystem — React's real advantage is the ecosystem around it (shadcn etc.).
- Related: Datastar (SSE + signals hypermedia), Unpoly (mature, progressive-enhancement-first).
- No silver bullet: hypermedia suits CRUD-heavy, server-owned-state apps; highly interactive client-state UIs (offline, rich editing, "3D flying") suit reactive frameworks.

**Rules of thumb**
- Fewer dependencies, closer to the platform = longer life and less maintenance.
- Learn how forms/links/HTTP work natively before reaching for a framework.
- Avoid dunking on others' tech choices; choose per problem.

**Relevance to Sam:** as a Python backend engineer, server-rendered HTML (Jinja/Django/FastAPI templates) + HTMX lets him build internal tools (data-quality review UIs for GeoNames matching, supplier mapping dashboards, admin panels for Trameter) with almost no JS and no build step, and AI-generated code stays reviewable in one language. For the embeddable Drive Button widget or highly interactive Tereka planner screens, a JS component may still be needed. The backward-compat/extension lessons apply directly to his supplier-integration adapters.

## Kevin Powell — Flexbox or grid - How to decide? (https://www.youtube.com/watch?v=3elGSZSWTbM)
**The real question is not 1D vs 2D, it is "content-out vs layout-in": use flexbox when items should size from their content (intrinsic sizing), grid when the parent should impose a rigid structure.**

**Mental models / first principles**
- Both can produce wrapped, apparently two-dimensional layouts. The difference is *control*: in a wrapped flex container each line is independent — a column in one row ignores the rows above and below. In grid, tracks are shared; a column is the same width in every row and items are locked into cells (spans allowed, no "in-between" positions; masonry is a future spec, subgrid came first).
- **Intrinsic sizing:** an element's natural size from its content. Flexbox's defaults respect it (items shrink-wrap their content, then squish when space runs out). Grid's defaults ignore it: the parent defines tracks and children fill them.
- Where the control lives: flexbox usually needs rules on the *children* (`flex-grow`, `flex-basis`, `flex: 1`) to change behaviour; grid is controlled almost entirely from the *parent*, so you can drop any children in.
- Choose the tool whose defaults already match your goal — the "path of least resistance" — instead of fighting it.

**Mechanisms / defaults**
- `display: flex` → `flex-direction: row`; every direct child becomes a column immediately, even if that overflows and causes horizontal scroll. Fix with `flex-wrap: wrap`.
- `display: grid` → `grid-auto-flow: row`; nothing visible changes (items stack). `grid-auto-flow: column` mimics flex roughly, with similar overflow risk.
- `grid-template-columns: repeat(5, 1fr)` = fixed column count. `repeat(auto-fit, minmax(300px, 1fr))` = responsive wrapping grid with no media queries, but every cell is equal width.
- On flex children: `flex-grow: 1` fills leftover space; `flex-basis: 33%` approximates three columns (last row items grow bigger); `flex: 1` gives near-equal columns but can still be uneven when children have differing padding, because of how the flex algorithm distributes space.
- `1fr` columns can also go unequal if one cell contains a very long unbreakable word (min-content); `minmax(0, 1fr)` is the stricter fix (note: Powell hints at minmax; `minmax(0,1fr)` is the standard fix).
- `gap` works in both.

**Worked examples**
- **Navigation bar** (items of varied length): flexbox wins — `display:flex; flex-wrap:wrap; gap` and each link keeps its own size and wraps naturally. Grid versions (auto-flow column, or auto-fit/minmax) give equal-width cells, uneven visual spacing and ugly wrapping.
- **Equal-width content columns/cards:** grid wins — `display:grid; grid-template-columns: repeat(3,1fr); gap` on the parent only. Flex needs parent + child rules and still fights intrinsic size (a long paragraph makes its column wider).
- **Mixing:** grid for the card layout, flex with wrap for the tag list inside each card. They nest freely; you are never locked into one.

**Decision checklist**
- Should each item be as big as its content? → flex.
- Should the parent dictate sizes and alignment across rows and columns? → grid.
- Page-level/large layouts, card grids, forms with aligned labels → grid. Toolbars, nav, button groups, tags, chips, centring one thing, "push this to the end" → flex.
- If you are writing lots of child rules to make flex items equal, switch to grid; if you are fighting grid to let items be their natural size, switch to flex.

**Failure modes**
- Flex without wrap → horizontal overflow on small screens (watch for this in injected widgets inside narrow blog columns).
- Using grid auto-fit for nav → rigid, awkward spacing.
- Expecting flex rows to align columns with each other.
- Powell's personal take: grid is easier to start with despite more properties, because there is less parent/child wrangling.

**Relevance to Sam:** the Drive Button widget lives in arbitrary host blog layouts of unknown width; buttons/chips inside it should be flex with wrap (content-sized, never overflow), while a results card list on Trameter or a Tereka dashboard should be grid with `auto-fit/minmax`. When reviewing AI CSS, a pile of `flex: 1`/`width: 33%` hacks is a smell suggesting grid.

## Kyle Cook (Web Dev Simplified) — React Server Components Change Everything (https://www.youtube.com/watch?v=rGPpQdbDbwo)
**Server Components run only on the server and ship rendered output, not code: data fetching becomes a plain `await`, secrets and heavy libraries stay server-side, and interactivity is confined to explicitly marked client islands.**

**Mental models / first principles**
- Classic client-rendered React is a two-step dance: ship HTML + JS → JS runs → `useEffect` fetches from a separately built API → manage loading/error/state → render. That requires building both an API and a frontend, and long effect/state code.
- Server Components (a React feature; Next.js App Router is the most complete implementation as of early 2024) collapse this into one step: the request hits the server, the component fetches/queries directly, and only the result is sent to the browser.
- The guarantee "never runs on the client" is what unlocks everything: async components, direct DB access, secrets, zero bundle cost — and also what forbids interactivity.
- Boundary rule: **where a component is imported decides where it runs.**

**Mechanisms (as demonstrated in Next.js)**
- Proof by `console.log`: a server component logs only in the server terminal, never the browser console. Client components log in the browser — and, in Next.js, also render once on the server on a hard page load (SSR for first paint/SEO), a Next behaviour, not React's.
- Server components can be `async`: `const todos = await fetch(...)` or `await db.todos.findMany()` directly in the component. Loading and error UI come from Next's `loading`/`error` route files (framework-specific).
- Security: request bodies, API keys and DB credentials in a server component never reach the client; only the returned markup/data does.
- Mark a client component with `"use client"` at the top of the file. Client components may use `onClick`, `useState`, `useEffect`, `window`, `localStorage`, `navigator`. Server components cannot use any hooks, event handlers or browser APIs.
- Nesting: server → client is allowed. Everything *imported by* a client component becomes client code too (a one-way door). To render a server component inside a client component, import it in a server parent and pass it as `children`/a prop; it is rendered on the server and the client component just places the result.

**Performance benefits claimed (three)**
1. **Shared server cache:** caching the rendered result/data on the server serves all users; client-side caches only help one user, so with client components you must cache at the API layer and still do the extra round trip.
2. **Smaller bundles:** libraries imported only in server components are never shipped. Demo: searching browser sources for a big helper found it on the client page and "no matches" on the server page.
3. **Faster complete load:** one response with data already in the HTML instead of HTML → JS → fetch → render (no incomplete-data flash).
- SEO: fully populated HTML on first response (search engines can run JS now, but this removes the dependency).

**Decisions & trade-offs**
- Default to server components; push `"use client"` down to the smallest interactive leaves (a button, a form widget), passing server-fetched data in as props.
- Costs: no interactivity or browser APIs in server components; the server must run per request (or cache); framework coupling — the lines between React and Next.js behaviour were blurry, other frameworks may differ.
- (note: the video does not cover security pitfalls later found in RSC/Server Actions — e.g. accidentally passing secret-containing objects as props to client components serialises them to the browser, and Server Actions are public HTTP endpoints needing auth checks. Also, in late 2025 critical RSC protocol vulnerabilities were disclosed and patched; keep React/Next updated.)

**Rules of thumb for reviewing AI-generated Next.js code**
- Is `"use client"` at the top of a high-level layout/page? That turns the whole subtree into client code and throws away the benefits — push it down.
- Any secret, DB client or heavy library imported into a file that is (or is imported by) a client component leaks into the bundle. Next offers a `server-only` import guard.
- Only pass serialisable, minimal data across the server→client boundary.
- `useEffect` + fetch for initial page data in an App Router project is usually a smell; fetch in the server component instead.

**Comparison with hypermedia (link to Carson Gross notes):** RSC and HTMX converge on "render on the server, send the result", differing in that RSC keeps React's component model and client hydration for islands, while HTMX sends plain HTML fragments and keeps client JS minimal.

**Relevance to Sam:** for Tereka or a Trameter UI on Next.js, supplier API keys (Viator, RateHawk) must only be used in server components/route handlers. For the Drive Button, which is injected into third-party blogs, RSC does not apply directly (it is a script on someone else's page), but the same principle holds: do the heavy work (LLM matching, supplier lookups) server-side and send the widget minimal rendered data.

## Maximilian Schwarzmüller (Academind) — How The Web Works - The Big Picture (https://www.youtube.com/watch?v=hJHvdBlSxug)
**Every web interaction is a client sending an HTTP request to a server found via DNS and getting a response back; only the payload format (HTML vs JSON) and who renders it change.**

**Mental models / first principles**
- The browser is essentially an interpreter: it receives source code (HTML, CSS, JS) and turns it into pixels. That code is either stored on the server as files (static) or generated per request (dynamic, e.g. a shop showing your cart).
- Servers are addressed by **IP addresses** (IPv4 like `172.56.180.5`, or longer IPv6). Domain names exist for humans. The first step of visiting a site is a **DNS lookup**: a DNS server acts as a giant dictionary mapping domain → IP; the browser then contacts that IP.
- The **request/response** pattern is universal: browsers, mobile apps and backend services all speak it. The "web" means any client talking to a remote server, not just websites.

**Mechanisms**
- A request is a data package containing: the URL, a **method/type** (e.g. GET to fetch, POST to submit a form/store data), optional **body** data, and **headers** (metadata). The response also has headers plus a body — for a traditional site, the HTML to render.
- Inspect both in DevTools → Network tab (headers, payloads, timing). This is the single most useful debugging habit.
- The browser auto-completes a bare domain (adds scheme etc.).
- **HTTP** standardises the shape of requests/responses; **HTTPS** is HTTP encrypted between browser and server, so someone sniffing the connection sees unreadable bytes. All modern sites should use HTTPS (note: HTTPS is TLS between endpoints — the video says "end-to-end", which is accurate for browser↔server but data is decrypted at the server and any proxy/CDN that terminates TLS).
- The three front-end languages: **HTML** = structure and meaning (heading, paragraph, input, button; a standardised format the browser parses); **CSS** = presentation; **JavaScript** = logic and anything that changes after load (dropdowns, tabs, overlays, sliding nav).
- Server-side: code in Node.js, PHP, Python etc. generates responses; **frameworks** handle repetitive plumbing like request parsing so you write business logic. Front-end frameworks similarly let you declare *what* should change while they handle *how* the DOM updates, enabling in-page updates without fetching a new page on every click.
- Apps (Instagram, Twitter) send the same kind of request but ask for **JSON** (via headers), and their own code converts that data into UI. Same pattern, different payload and different renderer.
- **WebSockets**: an established persistent connection over which the server can push messages without a client request — for real-time features.

**Decisions & trade-offs**
- Static files vs dynamic generation: static is simpler and cacheable; dynamic is needed for per-user content.
- HTML responses (browser renders) vs JSON responses (client code renders): this is the root of the server-rendering vs SPA debate covered by Carson Gross and Kyle Cook in this group.
- Request/response polling vs WebSockets for real-time: push needs a long-lived connection and more infrastructure.

**Rules of thumb**
- When something "doesn't load", walk the chain: DNS → connection/TLS → request (method, URL, headers, body) → response (status, headers, body) → browser parse/render → JS execution. Check the Network tab before reading code.
- Use correct methods: GET for reading (safe, cacheable), POST/PUT/PATCH/DELETE for changes.
- Never serve anything over plain HTTP.

**What the video does not cover (gaps to fill elsewhere):** TCP/TLS handshakes and their latency cost, status codes, caching headers, CDNs, and the browser's parse → DOM/CSSOM → layout → paint pipeline. It is a big-picture intro only.

**Relevance to Sam:** Sam already lives on the server side of this diagram (supplier APIs returning JSON). The key frontend insight: the Drive Button script runs inside a reader's browser on someone else's blog, so every request it makes is a cross-origin request from an untrusted client — it needs HTTPS, CORS configured on Trameter's API, and no secrets in the client.

## Jeff Delaney (Beyond Fireship) — The ultimate guide to web performance (https://www.youtube.com/watch?v=0fONene3OIA)
**Optimise initial load against Core Web Vitals — loading (LCP), responsiveness (FID, now INP) and visual stability (CLS) — and measure them per element and per page with real tools, not guesswork.**

**Mental models / first principles**
- Users bounce from slow pages; slowness costs money and also SEO ranking. Initial page load is where you win or lose attention.
- Each vital measures a different failure: "is the main content visible yet?", "does it react when I tap it?", "does it stay still?"

**The metrics (thresholds as stated)**
- **LCP — Largest Contentful Paint** (loading): time until the largest visible element (often a hero image/video/heading) renders. First Contentful Paint is the first anything; LCP is when the main thing arrives. Good ≤ 2.5 s, poor > 4 s.
- **FID — First Input Delay** (interactivity): delay between the user's first interaction (e.g. click) and the browser starting to run its event handlers. Good ≤ 100 ms, poor ≥ 300 ms. (note: FID was replaced by **INP — Interaction to Next Paint** as a Core Web Vital in March 2024; INP covers all interactions until the next frame paints; good ≤ 200 ms, poor > 500 ms. Same root cause and fixes: long main-thread JS tasks.)
- **CLS — Cumulative Layout Shift** (visual stability): elements jumping unexpectedly. (Thresholds not given in the video; standard: good ≤ 0.1, poor > 0.25.)

**Mechanisms & fixes**
- *LCP:* read the DevTools Network waterfall; wide bars are bottlenecks. Fixes:
  - Shrink resources: compress images, use modern formats (WebP; AVIF also), load the minimum number of fonts/weights.
  - Serve from a **CDN** (Vercel/Firebase do this automatically).
  - Avoid **render-blocking JavaScript** before main content. This is why server-rendered frameworks (Next.js) gained over pure client SPAs: SPAs must download and run JS before showing the main content.
  - Prioritise: `<link rel="preload">` so critical assets are discovered early; `fetchpriority="high"` on the hero image, `"low"` for below-the-fold images.
  - The Web Vitals extension breaks LCP into sub-parts such as **time to first byte** and **element render delay**, telling you whether the server or the front end is the bottleneck.
- *FID/INP:* the only real fix is less main-thread JavaScript execution. Tools: Web Workers, Partytown (moves third-party scripts into a worker), framework lazy loading (`React.lazy`), or resumable frameworks like Qwik that aggressively lazy-load.
- *CLS:* always give images `width` and `height` (or CSS `aspect-ratio`) so space is reserved before they load; for art-directed responsive images use `srcset`/`<picture>`. Other causes: injected ads/content and over-ambitious animations (animate `transform`/`opacity`, not layout properties).

**Tools**
- **Lighthouse** in Chrome DevTools: lab report for one page.
- **Web Vitals Chrome extension** with console logging: logs LCP (and the exact element responsible), CLS (the DOM nodes that shifted) and input delay per interaction — pinpoints what to fix.
- **Unlighthouse** (`npx unlighthouse --site <url>`): runs Lighthouse on every page of a site in parallel, finds unexpected slow pages; has an SDK/CI integration.
- Real-world scan: Google, Reddit and other big sites averaged ~90; Amazon fine overall but had CLS issues; astro.build was near-perfect (Astro ships little JS by default).
- (note: Lighthouse is lab data; Google ranks on **field data** from real Chrome users (CrUX / Search Console). Use both; INP especially needs field data since it depends on real interactions.)

**Decisions & trade-offs**
- SPA vs server-rendered: SPAs pay an LCP and interactivity cost up front; server rendering (Next.js, Astro, htmx-style) sends content first.
- Every third-party script (analytics, ads, chat widgets) costs main-thread time and may cause layout shift.

**Checklist for reviewing a page/AI-written frontend**
- Hero image: compressed, modern format, sized, `fetchpriority="high"`, not lazy-loaded; below-fold images `loading="lazy"`.
- All images/iframes/embeds have dimensions or reserved space.
- Minimal fonts; no render-blocking scripts in `<head>` (`defer`/`async`).
- Static assets via CDN with caching.
- Measure with the extension + Lighthouse; scan all pages with Unlighthouse; gate in CI.

**Relevance to Sam:** the Drive Button is a third-party script injected into bloggers' pages, so it directly affects *their* Core Web Vitals — bloggers will remove it if it hurts SEO. It must load async/deferred, be tiny, never block render, reserve its own space (fixed min-height/aspect) so it causes zero CLS when buttons appear, and keep click handlers cheap (good INP). This is a product-adoption risk, not just a technical nicety.

