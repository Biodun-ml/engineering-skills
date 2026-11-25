---
name: frontend-fundamentals-first-principles
description: First-principles frontend playbook from 13 videos (Kevin Powell, Matt Pocock, Philip Roberts, Lydia Hallie, Carson Gross, Kyle Cook, Jeff Delaney, freeCodeCamp a11y). Use when writing, reviewing or choosing a stack for UI, widgets, HTML/CSS/TS, React or htmx, or fixing slow or inaccessible pages.
---

# Frontend fundamentals from first principles

This skill turns 13 videos into frontend judgment for a backend/data engineer who mostly
*reviews* AI-written UI code. It covers how the browser turns bytes into pixels, the event
loop, CSS layout, accessibility, TypeScript, Core Web Vitals and where to render. This file
holds the procedures, principles and checklists. `references/` holds detailed notes for every
video, each credited to its teacher.

Use it whenever a browser is involved, even if nobody says "frontend": an embeddable script on
someone else's site, a form, a dashboard, an internal admin tool, "the page is slow", "the
button doesn't work on mobile", choosing React vs server templates, or reviewing a Claude- or
Lovable-generated UI.

**How to use it**
1. Pick the procedure that fits: choose a stack (section 1), build an embeddable widget
   (section 2), or review AI-written frontend code (section 5 checklists).
2. Use the principles in section 3 to explain *why* a change matters, crediting the teacher.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each
   section ends with a "Relevance to Sam" note: Sam builds the Drive Button (a script that
   injects bookable trip buttons into third-party travel blogs), Tereka (a free, zero-server
   money planner for Nigerians, used on low-end Android over slow networks), the Riselist
   agency site and Trameter UIs. Use those parts only in that context. Lines marked
   "(note: …)" and those Sam notes are helper additions, not the speaker's words. If
   `references/` is missing, the full notes live in the user's private GitHub repo
   `Biodun-ml/engineering-skills` (folder `frontend-fundamentals-first-principles/references/`);
   clone it when you need depth.
4. Browser support, framework versions and metric thresholds move. Check current ones (MDN,
   caniuse, web.dev) before relying on a specific API in production.

---

## 1. Procedure: pick a stack for a small product

1. **Ask who owns the state.** If the server owns it (bookings, listings, admin records,
   CRUD), the page is a view of server data: render HTML on the server. If the client owns it
   (offline drafts, rich editing, instant recalculation as you type), you need client JS.
   *(Carson Gross — HTMX in 2025; Kyle Cook — RSC)*
2. **Default to server-rendered HTML.** Finished HTML in one round trip paints sooner than
   HTML → JS → fetch → render, and that order is what LCP measures. *(Hitesh Choudhary;
   Kyle Cook; Jeff Delaney)*
3. **Choose the lightest tool that fits:**
   - Mostly static content (marketing, docs, blog) → plain HTML/CSS or a static-site
     generator that ships near-zero JS (Delaney's scan found astro.build near-perfect).
   - CRUD and internal tools with a Python backend → server templates (Jinja/Django/FastAPI)
     + htmx. One language, no build step, AI output stays reviewable. *(Gross)*
   - Rich client interactivity or a team already on React → React, server-first (Server
     Components), with `"use client"` pushed down to small leaves. *(Cook)*
   - Embeddable script on someone else's page → no framework, or a tiny one (section 2).
4. **Never rewrite a working app for fashion.** Trial a new approach on a neglected internal
   tool first. *(Gross)*
5. **Count dependencies as future work.** Every package is upgrades, conflicts and possible
   rewrites; closer to the platform lasts longer (the "100-year web service" idea). *(Gross)*
6. **Set the guardrails on day one:** TypeScript `strict`, `tsc --noEmit` in CI, Lighthouse/axe
   in CI, a performance budget, HTTPS only. *(Pocock; Delaney; Frederik; Schwarzmüller)*

## 2. Procedure: build and ship an embeddable widget (the Drive Button)

A third-party script runs on the host's main thread, inside the host's CSS, with the host's
users. Bloggers will remove it if it hurts their SEO, so performance is an adoption risk, not
a nicety. *(Delaney, applied to Sam)*

1. **Load without blocking.** One small `async` script tag; no `document.write`; no plain
   `<script>` in `<head>`. *(Choudhary)*
2. **Reserve space before content arrives** (fixed `min-height` or `aspect-ratio` on the
   placeholder), so buttons appearing cause zero CLS. *(Delaney)*
3. **Do the heavy work on the server.** LLM matching and supplier lookups happen in Trameter's
   backend; the widget receives a small, already-decided payload. Never scan a long article
   for place names on the host's main thread. *(Roberts; Cook's principle applied)*
4. **Keep handlers cheap** so clicks paint within 200 ms (INP). Break any unavoidable work
   into tasks that yield (section 3.2). *(Delaney; Hallie)*
5. **Isolate styles both ways** (Shadow DOM or an iframe, see box in section 7) and use
   `flex-wrap: wrap` + `gap` so buttons wrap in narrow blog columns instead of overflowing.
   *(Powell; Choudhary)*
6. **Be accessible inside a page you don't control:** real `<a href>` (navigates to booking)
   or `<button>` (acts), names that stand alone ("Book this Lisbon trip", not "Go"), no `h1`,
   your own background so contrast is guaranteed. *(Frederik)*
7. **Treat every request as cross-origin from an untrusted client:** HTTPS, CORS configured on
   Trameter's API, no secrets or supplier keys in the script. *(Schwarzmüller; Cook)*
8. **Validate the payload at runtime** before rendering; TS types on a fetch result are a
   promise, not a check. *(Pocock)*
9. **Measure on real host pages,** not a blank test page: Web Vitals extension to see whether
   your element is the LCP or a CLS source, and field data once installed. *(Delaney)*

---

## 3. Principles that recur across the teachers

### 3.1 How the browser turns bytes into pixels
- **HTML is parsed, never run.** Bytes → characters (charset) → tokens → nodes → DOM; the same
  for CSS → CSSOM; DOM + CSSOM → render tree → layout (geometry) → paint. Nothing is visible
  until paint. Declare `<meta charset="utf-8">` first. *(Choudhary)* (note: modern engines add a
  GPU compositing step after paint; animate `transform`/`opacity`, which can skip layout and
  paint.)
- **Classic scripts block the parser, and pending CSS blocks scripts.** So big CSS delays
  everything. Put CSS in `<head>`, keep it lean. *(Choudhary)*
- **`defer` vs `async`:** `defer` runs after HTML parsing, in document order — default for your
  own app code. `async` runs as soon as downloaded, in any order — for independent scripts like
  analytics and embeds. *(Choudhary, corrected in notes; he blurs the two)* (note: `type="module"`
  scripts are deferred by default.)
- **DevTools Elements shows the DOM after scripts ran,** not the source HTML. Use View Source
  for what the server sent. *(Choudhary)*
- **Every interaction is DNS → HTTP request (method, headers, body) → response.** When
  something "doesn't load", walk that chain in the Network tab before reading code. GET reads,
  POST/PUT/PATCH/DELETE change. *(Schwarzmüller)*
- **The browser builds a second tree from your markup: the accessibility tree.** Assistive tech
  reads that, not your pixels. *(Frederik)*

### 3.2 The event loop
- **One thread, one call stack.** While JS runs, the browser cannot render, scroll or respond
  to clicks. "Blocking" just means slow code sitting on the stack. *(Philip Roberts)*
- **Async is the host's job.** `setTimeout`, `fetch`, DOM events and geolocation are Web APIs;
  the JS call registers a callback and returns, the browser does the work. *(Roberts; Lydia
  Hallie)* Same shape in Node with libuv.
- **Two queues, strict order:** sync code → drain *all* microtasks (`.then`, code after
  `await`, `queueMicrotask`, MutationObserver) → one task (timers, events) → microtasks again.
  Rendering gets a turn only when the stack is clear (~every 16.6 ms at 60 fps). *(Hallie;
  Roberts)* Quiz: `Promise.then(1)`, `setTimeout(2,10)`, `queueMicrotask(3 → queues 4)`,
  `log(5)` prints 5 1 3 4 2.
- **Recursive microtasks starve rendering** exactly like a sync loop. To let the page paint
  between chunks, schedule a *task* (`setTimeout`, `MessageChannel`, `scheduler.yield()`),
  never a chain of awaits on resolved promises. *(Hallie)* (note: `scheduler.yield()` is in
  Chromium 129+ and Firefox 142+ but not Safari as of late 2026: feature-detect and fall back
  to `setTimeout`.)
- **Timer delays are minimums:** the delay is time until *queued*. `setTimeout(fn, 0)` means
  "after the current stack clears". *(Roberts; Hallie)*
- **High-frequency events flood the queue:** debounce or throttle scroll, resize and input
  handlers. *(Roberts)*
- **Not every callback is async:** `forEach` runs its callback synchronously. *(Roberts)*
- **Long work goes elsewhere:** chunk it, move it to a Web Worker, or move it to the server.
  Never synchronous XHR. *(Roberts)*

### 3.3 CSS layout
- **Decision rule: content-out → flex; layout-in → grid.** Should each item be as big as its
  content (nav links, chips, button groups, tags, centring one thing)? Flex. Should the parent
  impose shared columns and rows (cards, page layout, forms with aligned labels)? Grid. Nest
  freely: grid for cards, flex-wrap for the tags inside each card. *(Kevin Powell — Flexbox or
  grid)* This replaces the older "1-D vs 2-D" rule, since wrapped flex looks 2-D too.
- **Smell test:** lots of `flex: 1` / `width: 33%` child hacks → you wanted grid. Fighting grid
  to let items be natural size → you wanted flex. *(Powell)*
- **Flex starts at max-content and shrinks toward min-content** (`flex-shrink: 1` default,
  `flex-grow: 0`). Grow/shrink are ratios, so plain `display:flex` gives *unequal* columns;
  `flex: 1` (basis 0) gives equal ones. Each wrapped row is independent. *(Powell — flexbox)*
- **`justify-content` only distributes leftover space.** It does nothing if items are
  shrinking, something grows, or a column container has no height (use `min-height`).
  `align-items` defaults to `stretch`. *(Powell)*
- **Flex without wrap overflows** small screens — critical for a widget in a narrow blog
  column. *(Powell)*
- **Grid: add complexity only when forced:** `display:grid` → `gap` → `repeat(n, 1fr)` → spans
  → explicit lines last. Let the implicit grid make rows; don't declare `grid-template-rows`
  without a reason. Prefer `span N` over brittle line numbers. *(Powell — CSS Grid)*
- **Uniform card grid without media queries:** `repeat(auto-fit, minmax(300px, 1fr))`. If a
  long unbreakable word widens a column, use `minmax(0, 1fr)`. *(Powell)*
- **`grid-template-areas`** lets each breakpoint redefine only the parent; add
  `grid-auto-columns: 1fr` to stop uneven auto-sized columns. *(Powell)*
- **Mobile-first:** single column base, add columns in `min-width` queries. Use `gap`, not
  margins, in both flex and grid. *(Powell)*
- **Visual placement does not change DOM order;** keyboard and screen readers follow the DOM,
  so keep source order meaningful. *(Powell, note in references)*

### 3.4 Accessibility
- **Native element first; everything else is a patch.** `<button>`, `<a href>`, `<label>`,
  landmarks and lists give roles, names and keyboard support for free. First rule of ARIA: don't
  use ARIA when HTML can do it. Link navigates; button acts. *(Frederik — freeCodeCamp a11y)*
- **Contrast (WCAG):** AA = 4.5:1 body text, 3:1 large text; AAA = 7:1. AA is the usual target;
  level A is the minimum, not "inaccessible" (the teacher misstates this). Check placeholder
  and footer text, the usual failures. *(Frederik, corrected)*
- **Never carry meaning by colour alone;** pair status colours with icon + text. *(Frederik)*
- **Names must stand alone:** screen-reader users pull up a list of links, so no "click here".
  Every input has `<label for>`; placeholder is an example, never the label; radio groups go in
  `fieldset`/`legend`. *(Frederik)*
- **Alt text:** informative images get a short description (no "image of"); decorative ones get
  `alt=""`. A missing alt may make readers announce the filename. *(Frederik)*
- **Card links:** link the heading/CTA and stretch its hit area with `::after { inset: 0 }`
  instead of wrapping the whole card in `<a>`. *(Frederik)*
- **Structure is navigation:** one `h1`, no skipped heading levels (headings are how
  screen-reader users mostly navigate), landmarks, real lists, font sizes in `rem` so user font
  settings work. *(Frederik)* (note: Safari/VoiceOver drops list semantics with
  `list-style: none`; add `role="list"`.)
- **Announce changes and move focus on purpose:** `aria-live="polite"` for status and result
  updates (`assertive` only for urgent ones); after submit or modal open, `.focus()` the next
  logical element; add a skip link that stays focusable. Overuse disorients. *(Frederik)*
  (note: a live region must already be in the DOM before its text changes, or it won't announce.)
- **Verify with tools and by hand:** Lighthouse, axe, the DevTools accessibility tree and
  contrast tooltip, then a keyboard-only pass and a real screen-reader pass. Automated scores
  miss things. *(Frederik)*

### 3.5 TypeScript
- **Types are erased at runtime.** TS catches bugs in the editor and CI only; API responses,
  `JSON.parse` and user input still need runtime validation (a schema validator, deriving the
  type from the schema). `as Booking` on a fetch result is an unchecked promise. *(Matt Pocock,
  both videos)*
- **TS is only as good as its strictness:** `"strict": true` plus `noUncheckedIndexedAccess`.
  `tsc` emits JS even with type errors by default, and the editor checks only open files, so
  run `tsc --noEmit` across the project in CI. *(Pocock)*
- **Escape hatches are lies to the checker:** `any` (which *spreads* to everything it touches),
  `as`, `as unknown as`, `@ts-ignore`, non-null `!`. Pocock compares `as` to CSS `!important`.
  Each one needs a reason. *(Pocock)*
- **Model states as discriminated unions,** not bags of optional fields:
  `{status:"pending"} | {status:"confirmed", ref:string} | {status:"failed", reason:string}`.
  Narrow with `typeof` or the discriminant. Prefer string-literal unions to `enum` (enums emit
  runtime code). *(Pocock)*
- **Annotate where data is created** (params, return types, object literals) so errors land at
  the mistake, not downstream; infer everywhere else. `type` vs `interface` for plain objects
  barely matters — be consistent. *(Pocock)*

### 3.6 Performance and Core Web Vitals
- **Three vitals, three questions:** is the main content visible (LCP ≤ 2.5 s good, > 4 s
  poor)? Does it react when I tap (INP ≤ 200 ms good, > 500 ms poor)? Does it stay still (CLS
  ≤ 0.1 good, > 0.25 poor)? *(Jeff Delaney; INP replaced FID in March 2024 and CLS thresholds
  come from web.dev — both helper corrections)*
- **LCP:** read the Network waterfall; compress images, AVIF/WebP, fewer fonts and weights, a
  CDN, no render-blocking JS before main content, `preload` critical assets,
  `fetchpriority="high"` on the hero (never lazy-load it). The Web Vitals extension splits LCP
  into TTFB vs render delay so you know if the server or the front end is slow. *(Delaney)*
- **INP:** the real fix is less main-thread JS: workers, lazy loading, smaller bundles, yielding
  in long handlers. *(Delaney; Hallie)*
- **CLS:** `width`/`height` or `aspect-ratio` on every image, iframe and embed; reserve space
  for injected content; animate `transform`/`opacity`, not layout properties. *(Delaney)*
- **Every third-party script costs the host** main-thread time and possible shifts.
  *(Delaney)*
- **Lab vs field:** Lighthouse is lab data on one page; Unlighthouse scans every page and can
  gate CI; Google ranks on field data from real Chrome users (CrUX/Search Console). INP in
  particular needs field data. *(Delaney, note)*

### 3.7 Where to render
- **Rendering location is the central architecture choice.** Server-rendered HTML (htmx
  hypermedia, React Server Components) sends finished content in one trip; client rendering
  adds steps that cost LCP and complexity. *(Gross; Cook; Delaney)*
- **Hypermedia:** any element, on any event, issues any HTTP method and swaps returned HTML into
  a CSS-selected target. Template fragments render one chunk for partial updates while
  keeping the page readable in one file — locality of behaviour. *(Gross)*
- **RSC:** where a component is imported decides where it runs. Server components can `await`
  the DB directly, keep secrets and heavy libraries off the client, and share a server cache.
  `"use client"` is a one-way door — everything it imports ships to the browser.
  `useEffect` + fetch for initial data in an App Router project is a smell. *(Cook)*
- **Library design (Gross):** get the core concept right, never break backward compatibility
  (add a config flag for old behaviour), say no to features, extend through events. Applies to
  Sam's supplier adapters as much as UI libraries.

---

## 4. Where teachers disagree, and how to resolve it

| Tension | Resolution |
|---|---|
| SSR + hydration (Choudhary) vs RSC islands (Cook) vs plain HTML + htmx, no hydration (Gross) | They agree on "server first"; they differ on client JS. Server owns the state and Python backend → htmx. Rich client state or an existing React team → RSC with small client leaves. Gross himself says no silver bullet: offline or rich editing needs a reactive framework. |
| Fewer dependencies (Gross) vs React's ecosystem (Gross concedes; Cook) | Ecosystem wins when you need many complex components fast (shadcn etc.); platform wins for longevity and CRUD. Count each dependency as maintenance. |
| One task queue (Roberts, 2014) vs two queues (Hallie) | Hallie supersedes: Roberts predates widespread Promises. Use Roberts for the big picture, Hallie for ordering. |
| FID (Delaney) vs INP | INP replaced FID in March 2024. Same root cause (long main-thread tasks), stricter scope: all interactions, not just the first. |
| "1-D vs 2-D" (common rule) vs "content-out vs layout-in" (Powell) | Use Powell's rule; wrapped flex can look 2-D but rows never align. |
| `enum` vs literal unions (Pocock vs his chat) | Literal unions. Enums emit runtime code and don't work with type-stripping. |
| Frederik's `div` with `role="switch"` + `tabindex` toggle | Contradicts his own first rule of ARIA. Prefer `<button role="switch" aria-checked>` (keyboard for free) or a styled `<input type="checkbox" role="switch">`. |
| Frederik pastes code into an LLM to check a11y | Fine as a first scan; not verification. Confirm with axe plus a keyboard and screen-reader pass. |

---

## 5. Checklists

**Review AI-written frontend code** (run first; these are the usual AI mistakes)
- [ ] `div`/`span` with `onClick` → should be `<button>` or `<a href>`. *(Frederik)*
- [ ] Inputs with placeholder but no `<label>`. *(Frederik)*
- [ ] `tsconfig` still `strict`; grep `any`, `as `, `as unknown as`, `@ts-ignore`, `!`. An AI
      that turned strict off to "fix" errors is a red flag. *(Pocock)*
- [ ] Fetch results cast with `as` instead of validated at runtime. *(Pocock)*
- [ ] States modelled as optional fields instead of a discriminated union. *(Pocock)*
- [ ] `"use client"` at a page/layout top; secrets, DB clients or heavy libs imported by client
      code; non-serialisable or secret-containing props crossing the boundary. *(Cook)*
- [ ] `innerHTML` / `dangerouslySetInnerHTML` with supplier, user or LLM text → XSS; use
      `textContent` or the framework's escaping. (reviewer addition)
- [ ] Margins instead of `gap`; `justify-content` on a heightless column; `width` and
      `flex-basis` both set; flex without wrap; percentage maths instead of `fr`. *(Powell)*
- [ ] Plain `<script>` in `<head>`; `async` app code that depends on order (should be
      `defer`). *(Choudhary)*
- [ ] Long sync loops or recursive `await` chains on the main thread; undebounced input/scroll
      handlers. *(Roberts; Hallie)*
- [ ] Images without dimensions; hero lazy-loaded; below-fold images not lazy. *(Delaney)*
- [ ] `useEffect` fetching initial data where the server could render it. *(Cook)*
- [ ] Code runs and *looks* right but nobody tabbed through it or checked the Network tab.

**Accessibility**
- [ ] Native elements; ARIA only where no element exists.
- [ ] Contrast ≥ 4.5:1 body text, 3:1 large text, including placeholders and footers.
- [ ] No colour-only meaning.
- [ ] Every input labelled; radio groups in `fieldset`/`legend`; errors announced and focus
      moved to the first error on submit.
- [ ] Link/button names stand alone; links distinguishable without colour.
- [ ] Alt on every image (`alt=""` if decorative).
- [ ] One `h1`, no skipped levels, landmarks, lists; `rem` font sizes.
- [ ] Visible focus on every interactive element; logical tab order = DOM order; skip link.
- [ ] Polite live region for status updates, present in the DOM before it changes.
- [ ] Lighthouse/axe in CI, plus manual keyboard and screen-reader passes.
- [ ] (WCAG 2.2 AA, reviewer addition) focus indicators and control borders ≥ 3:1; touch
      targets at least 24×24 CSS px (aim for ~44–48 px on mobile).

**Performance**
- [ ] LCP element identified (Web Vitals extension); hero compressed, sized,
      `fetchpriority="high"`, not lazy.
- [ ] No render-blocking scripts; CSS lean; fonts minimal.
- [ ] Every image/iframe/embed reserves space; animations use `transform`/`opacity`.
- [ ] No long tasks in click/input handlers; heavy work in a worker or on the server.
- [ ] Static assets on a CDN with long cache headers.
- [ ] Tested on a throttled mobile profile (DevTools CPU slowdown + slow network), not only on
      a fast laptop.
- [ ] Unlighthouse over every page; field data (CrUX/Search Console) once live.
- [ ] A JS budget written down and checked per build (reviewer addition).

---

## 6. Failure modes to hunt for

- Div-buttons and placeholder-only inputs (the two most common AI a11y failures).
- "Fast JS" that can't help because CSS or a parser-blocking script delays paint.
- Turning off `strict`, or `as`-casting supplier JSON, to make errors go away.
- A client-rendered SPA where server HTML would do: blank shell until the bundle runs.
- `"use client"` at the root, throwing away every RSC benefit.
- Recursive microtasks or one long `async` loop that never lets the page paint.
- An embedded widget that blocks the host, shifts the article, or inherits the host's CSS.
- Wrapping whole cards in `<a>`; meaning conveyed by a coloured dot alone.
- Grid placement that reorders visually so tab order no longer matches what users see.
- Secrets or supplier keys in anything shipped to the browser.
- Trusting Lighthouse alone; never looking at field data.

---

## 7. Not covered by the sources, but know this

These points are reviewer additions (senior frontend practice as of late 2026), not from the
13 videos. Verify versions before depending on them.

- **Widget isolation: Shadow DOM vs iframe.** Shadow DOM keeps host CSS selectors out and your
  CSS in, is light, and lives in the host's document, but inherited properties (font, colour,
  line-height) still flow in — reset them on `:host` with `all: initial` then set your own.
  An iframe gives full isolation of CSS, JS and data, but needs sizing via `postMessage` and
  costs a whole document. Rule: inline buttons in Shadow DOM; anything with personal or payment
  data (the booking flow) in an iframe or a new tab on Trameter's domain, never in the host's
  page where host scripts can read it.
- **`rem` inside a widget follows the *host's* root size,** and some blogs set
  `html { font-size: 62.5% }`. Set `:host { font-size: medium }` (the user's default size) and
  use `em` inside, so you still respect user settings without trusting the host.
- **Third-party constraints:** hosts may run a Content Security Policy that blocks your script
  or injected `<style>`; document the domains to allow. Don't rely on third-party cookies
  (Safari and Firefox block them by default); use your own domain for logged-in flows.
  Version the script URL, serve with long cache plus a short-cached loader, and offer an
  SRI-pinned versioned URL for hosts that want it.
- **Bundle budgets:** write a number down and fail the build when exceeded. A widget should be
  a few KB to low tens of KB compressed; an app's critical-path JS should stay small enough to
  parse quickly on a low-end Android (parse/execute cost, not just download, is the problem).
  Preact or Lit, or vanilla TS, beat full React for an embed.
- **Forms and validation:** start with native attributes (`required`, `type="email"`,
  `inputmode="decimal"` for money on mobile keyboards, `autocomplete`), link errors with
  `aria-describedby` and mark `aria-invalid`, and always revalidate on the server. Client
  validation is UX; server validation is security.
- **State management basics:** separate server state (cache it with a query library or let
  htmx/RSC refetch) from UI state (local component state) from URL state (filters, tabs —
  shareable and survives reloads). Reach for a global store last.
- **PWA and offline for low connectivity:** a service worker can cache the app shell
  (cache-first for static assets, network-first for data) and queue writes made offline,
  syncing when the network returns. Use IndexedDB, not `localStorage`, for anything sizeable.
  Safari may evict script-written storage for sites not used in about a week unless installed
  to the home screen, so never treat browser storage as the only copy.
- **i18n and NGN:** format with `Intl.NumberFormat("en-NG", { style: "currency", currency:
  "NGN" })` (→ ₦1,234.50) and `Intl.DateTimeFormat`; store money as integer kobo, never floats.
  Let the locale, not string concatenation, decide separators.
- **Image optimisation:** `srcset`/`sizes` so phones don't download desktop images; AVIF/WebP
  with fallback; explicit dimensions; `loading="lazy"` below the fold only; an image CDN or a
  build-time pipeline.
- **Tailwind:** fine with AI codegen because styles sit next to markup (locality of
  behaviour). v4 moved configuration into CSS. Review the *output*: long class strings still
  hide div-buttons and missing labels.
- **Vite:** the default dev server and bundler for non-framework and React SPA projects; its
  library mode builds a single-file widget bundle. Check bundle size with a visualiser plugin.
- **htmx 4.0 (released August 2026)** moved to `fetch`, made attribute inheritance explicit
  (`hx-headers:inherited` — a CSRF header silently stops reaching children without it),
  renamed events (`htmx:after:request`) and added a default request timeout. 2.x stays
  supported; run the upgrade checker before moving.
- **RSC security:** Server Actions are public HTTP endpoints — authorise inside each one. Never
  pass objects containing secrets as props to client components (they are serialised to the
  browser). Critical RSC protocol vulnerabilities were disclosed and patched in late 2025, so
  keep React/Next patched.

---

## 8. Applying this to Sam's work

- **Drive Button:** follow section 2 exactly. Vanilla TS or Preact, built with Vite library
  mode into one `async` script; Shadow DOM for the inline buttons, booking on Trameter's
  domain; reserved height; `flex-wrap` + `gap`; `textContent` for LLM-extracted titles (they
  originate in third-party blog text); runtime-validated payload; CORS on the API. Measure
  CLS/INP on three real partner blogs before each release. The Creator Studio (internal, SSE
  updates) is a natural htmx + server templates candidate if it is ever rebuilt.
- **Tereka:** mobile-first for low-end Android on slow, costly data. Its zero-server, data-in-
  the-user's-Google-Sheet design rules out RSC and server-rendered htmx, so ship a static,
  prerendered shell (fast LCP without a server) plus a small client app; PWA app shell with an
  offline queue for check-ins; `Intl` NGN formatting with integer kobo; grid for the cup
  dashboard, flex for chips; big tap targets, high contrast for outdoor screens, plain labels;
  test with CPU and network throttling every release. Debounce recalculation as users type.
- **Riselist site:** marketing content with near-zero interactivity → static HTML, almost no
  JS, compressed hero with `fetchpriority="high"`, contrast and alt text checked. It sells to
  international brands, so a perfect Lighthouse score is cheap credibility.
- **Trameter UI and internal tools:** server-first; supplier keys (Viator, RateHawk) only in
  server code; booking status as a discriminated union validated at the boundary; data-quality
  review screens for GeoNames matching as Python templates + htmx; `auto-fit/minmax` grid for
  results cards.

---

## 9. Where to look in `references/`

| File | What it covers |
|---|---|
| `fe-1-frederik-powell-choudhary-pocock-roberts-hallie.md` | Frederik (freeCodeCamp) full accessibility tutorial; Kevin Powell flexbox; Hitesh Choudhary how a browser works and the critical rendering path; Matt Pocock TypeScript speedrun; Philip Roberts event loop (JSConf EU 2014); Lydia Hallie event loop, Web APIs and microtask queue |
| `fe-2-pocock-powell-gross-cook-schwarzmller-delaney.md` | Matt Pocock TypeScript crash course (annotation location, discriminated unions, `as`/`any`); Kevin Powell CSS Grid; Carson Gross HTMX in 2025; Kevin Powell flexbox or grid; Kyle Cook React Server Components; Maximilian Schwarzmüller how the web works; Jeff Delaney web performance and Core Web Vitals |
| `INDEX.md` | Line number of every video section |
