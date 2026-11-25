---
name: api-payments-security-first-principles
description: Defensive API, web and payment security playbook from 17 talks (Stanford CS253, OWASP Top 10s, Corey Ball, OktaDev, Computerphile, Arpit Bhayani, PCI DSS). Use when building or reviewing any endpoint, login, session, payment, booking, webhook, URL fetcher, LLM pipeline or secret.
---

# API, web and payment security from first principles

This skill turns 17 talks and lectures into defensive judgment for a solo backend/AI engineer.
This file holds the procedure, the principles the teachers keep coming back to, and
checklists. `references/` (when present) holds detailed notes for every video, each credited
to its teacher. Everything here is defensive: attacks are described only as far as you need to
defend and test your own systems.

Use it whenever code touches trust, even if nobody says "security": a new route, a login or
"connect your account" flow, cookies, anything with an ID in the URL, money or bookings,
webhooks, a feature that fetches a URL, an LLM reading web pages, an API key, a new
dependency, or reviewing code an AI wrote.

**How to use it**
1. Follow the "secure a new endpoint" procedure for any new route or integration.
2. Run the matching checklists (section 3) against the design, diff or PR.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds
   supplier integrations (Viator, RateHawk), booking flows, an LLM pipeline that fetches travel
   blog pages and turns them into bookable trips, a money planner (Tereka) and an agency site
   (Riselist), mostly with AI-written code. Use those parts only in that context. Lines marked
   "(note: …)" and those Sam notes are helper additions, not the speaker's words. If
   `references/` is missing, the full notes live in the user's private GitHub repo
   `Biodun-ml/engineering-skills` (folder `api-payments-security-first-principles/references/`);
   clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Standards move (OWASP lists, NIST, PCI DSS, OAuth BCPs). Section 7 holds reviewer
   corrections and current facts as of late 2026; check again before quoting a version number.

---

## 1. Procedure: secure a new API endpoint

1. **Threat-model it in five minutes.** Who calls it (anonymous, user, admin, partner, cron)?
   What object, money or data does it touch? What happens if it is called 1,000 times a
   second, by the wrong user, with edited fields, or replayed? Write the business invariants
   down ("amount > 0", "refund ≤ charge", "price from the server"), because scanners and AI
   coders will not invent them. Use STRIDE if you want a prompt. *(Aikido — A06 Insecure Design;
   Corey Ball — business logic; IBM/Crume — threat-model at design time)*
2. **Authenticate with a boring, standard mechanism.** Browser app: opaque random session ID
   in a hardened cookie. Third-party login: OIDC code flow (+ PKCE). Machine-to-machine: client
   credentials or a scoped API key. Prefer a managed identity provider to homegrown auth.
   *(Feross L03/L13; Barbettini/OktaDev; Aikido A07)*
3. **Authorise at three levels, on every request, deny by default** in one central module:
   - *Object (BOLA/API1):* does this caller own or have rights on *this* ID? Filter by owner
     in the query (`WHERE id = ? AND owner_id = ?`), not after fetching.
   - *Function (BFLA/API5):* may this role call this action/verb? Admin routes inherit a
     base that checks role; check tenant/org boundaries.
   - *Property (BOPLA/API3):* allow-list writable fields per endpoint and role; return an
     explicit response schema (DTO), never the whole ORM object.
   Return 401 for "not logged in", 403 or 404 for "not yours". *(Corey Ball; Paxton-Fear;
   Aikido A01)*
4. **Validate input as data, never as code.** Typed schema with max lengths, array sizes,
   nesting depth and numeric ranges (no negative amounts). Parameterised SQL everywhere,
   allow-listed identifiers for table/column names, no shell strings. *(Computerphile/Pound;
   Corey Ball; Feross L01)*
5. **Never trust client-sent price, currency, status, role or totals.** Recompute from the
   server-side quote. *(Paxton-Fear — API3 seat upgrade; Feross L01)*
6. **Bound the cost.** Rate limit per user and per IP (stricter on auth/reset), page-size caps,
   payload size, timeouts, and spend caps on paid downstream calls (LLM, SMS, supplier
   quota). For abusable flows (holds, coupons, sign-ups) add bot/timing signals and expiring
   holds. *(Corey Ball API4/API6; Feross L13)*
7. **Make state changes safe to retry.** Mutations never on GET. Money/booking/irreversible
   actions take an idempotency key enforced by a unique constraint. Cookie-authenticated
   state changes get SameSite + a CSRF token or Origin/`Sec-Fetch-Site` check.
   *(Arpit Bhayani; Feross L04)*
8. **If it fetches a URL or consumes a third-party API, treat the response as hostile**
   (sections 3D and API10). *(McKenzie/Aikido SSRF; Corey Ball API7/API10)*
9. **Fail closed and quietly.** Uniform, schema-enforced errors; no stack traces, SQL errors or
   `X-Powered-By`; same status, message and timing for "unknown user" and "wrong password".
   *(Corey Ball API8; Feross L13)*
10. **Log and alert** on auth failures, denials and validation errors, with secrets, tokens and
    card data redacted. *(Aikido A09; Corey Ball; Feross L13)*
11. **Prove it with tests before shipping:** two-account replay (user A requests/edits B's
    object → must fail), a lower-role account calling admin functions, mass-assignment test
    (send `role`, `price`, `paid` → ignored), double-submit test for idempotency. Block deploys
    that fail them. *(Paxton-Fear; Corey Ball)*
12. **Register it in your inventory** (host, version, who can reach it, what data) and retire
    the old version. *(Corey Ball API9; Paxton-Fear API9)*

---

## 2. Principles that recur across the teachers

### Mindset
- **Assume one layer has already failed.** Design so the system holds when the attacker has
  the password, a DB dump or an XSS foothold: defence in depth. Correctness and security are
  separate properties; attackers need one hole and are paid professionals. *(Feross L01, L13)*
- **Never trust the client.** Anyone can curl your API with requests your UI never sends.
  Hiding a button is not access control; anything sent to the browser can be read and edited.
  *(Feross L01, L03; Corey Ball)*
- **Keep data from becoming code.** SQLi, command injection, XSS and SSRF share one root:
  attacker data reaching an interpreter or a fetcher. Fix structurally, not by filtering.
  *(Feross L01; Pound; McKenzie)*
- **Composition creates holes.** Every plugin, proxy, extension and integration brings its own
  security model; model each one. *(Charlie Reis; Paxton-Fear API10)*
- **Shift left.** Bugs found late cost far more (Crume cites up to ~640x; see section 7).
  Run SAST + DAST + dependency and secret scanning; they catch different classes. *(Crume)*
- **Attackers learn from every oracle:** verbose errors, headers, status codes and timing
  (25 ms vs 510 ms reveals which records exist). *(Corey Ball API8; Pound — blind SQLi)*

### Browser model
- **Origin = scheme + host + port; that is the boundary SOP enforces.** SOP blocks *reading*
  cross-origin responses, not *sending* forms, images or navigations. *(Feross L04; Rehberger)*
- **Cookies scope by domain, not origin; `Path` is not a boundary; same-site ≠ same-origin.**
  Sibling subdomains and other ports on the same host are same-site, so SameSite does not
  protect them from each other. Put login and sensitive apps on their own subdomain; set
  `Path=/` explicitly; avoid `Domain=`. *(Feross L03, L04; Rehberger)*
- **Ambient authority is the root of CSRF.** The browser attaches cookies to any request to
  your site, whoever triggered it; switching to POST does not fix it. Defend with
  `SameSite=Lax` (Strict for very sensitive apps) plus a CSRF token or Origin/`Sec-Fetch-Site`
  check on every state change. *(Feross L03, L04; Reis — Netflix 2006)*
- **Script in your page can do anything the user can.** XSS is the worst browser bug; encode
  output for its exact context, never reflect URLs or params into script. Declare what may run:
  CSP with nonces/hashes, SRI for third-party scripts, `frame-ancestors` (or
  `X-Frame-Options`) against clickjacking, correct `Content-Type` + `nosniff`.
  *(Reis; Rehberger)*
- **HTTPS on every page, plus HSTS.** In-flight modification is real: ~1.3% of page loads in
  Reis's 2008 study were rewritten, ~90% injecting JavaScript; Firesheep stole cookies sent in
  clear. *(Reis; Feross L03; Rehberger)*
- **Cross-origin talk is opt-in and origin-checked:** `postMessage` with explicit target
  origin and an `event.origin` allow-list; deliberate CORS; never `document.domain`. Keep
  tokens and PII out of URLs (`Referer`, logs). *(Feross L04; Rehberger)*

### Sessions and authentication
- **Authentication (who) and authorisation (what) are separate.** Hand out an opaque session
  ID and look permissions up per request; don't mint tokens that encode permissions you can't
  revoke. *(Feross L13)*
- **Session = ~128-bit CSPRNG ID mapped to a server-side row**, deleted on logout *and* on
  password change. Sequential IDs are enumerable; signed self-contained cookies can't be
  revoked. Cookie: `HttpOnly; Secure; SameSite=Lax; Path=/`, explicit modest expiry
  (rolling ~30 days). *(Feross L03, L04)*
- **Bearer tokens and cookies are the new passwords,** because stealing one skips MFA. Never in
  localStorage, never in logs. *(MacDonald/CNCF; Feross L13)*
- **Don't build auth if you can avoid it;** if you must: argon2id (bcrypt caps input at 72
  bytes), length over complexity (15+ chars password-only, see section 7), breached-password
  check, no forced rotation or paste-blocking, rate limits per account and per IP (~10x the
  most extreme real user), MFA mandatory for financial apps. *(Aikido A07; Feross L13)*
- **Leak nothing.** Same message, status *and timing* for unknown user, wrong password and
  locked account, on login, reset and signup (hash anyway on the "no user" path). *(Feross L13)*
- **Treat recovery endpoints as login endpoints.** A 6-digit reset code is only 10^6 guesses;
  rate-limit and lock out harder than general limits. *(Corey Ball API2)*
- **Re-authenticate ("sudo mode") before sensitive changes:** email, password, MFA phone,
  payout details, API keys. *(Feross L13; Corey Ball)*
- **MFA:** Microsoft claims it blocks 99.9% of account compromises; TOTP secrets are shared
  secrets, so encrypt them at rest. CAPTCHAs are weak (~$0.50 per 1000 solved by farms).
  *(Feross L13)*

### OAuth 2.0, OIDC and JWT
- **OAuth = delegated authorisation; OIDC = authentication** (scope `openid` → ID token).
  An access token or API key is not proof of who the user is. *(Barbettini; Corey Ball)*
- **Use the authorization code flow + PKCE** (RFC 9700: PKCE for every client). The browser
  sees only a useless code; the backend swaps it with the client secret. Implicit and password
  grants are deprecated. Unguessable `state` bound to the session (login CSRF), `nonce` checked
  in the ID token, exact redirect URIs, minimum scopes. *(Barbettini, with notes)*
- **Client secret, access and refresh tokens live on the back channel only.** For SPAs use a
  backend-for-frontend; store refresh tokens encrypted. *(Barbettini)*
- **A JWT is a readable signed claim; verify it fully.** Pin the algorithm on the verifier
  (never trust the header's `alg`, reject `none`), verify the signature *before* reading
  claims, then `exp`, `nbf`/`iat`, `iss`, `aud` (and `nonce` for ID tokens). Use a maintained
  library: in PyJWT pass `algorithms=[...]`, `audience`, `issuer`. *(MacDonald; Corey Ball)*
- **HS256 secrets ≥ 256 random bits** (a 4-character secret fell in ~10 s); prefer RS256/ES256
  with a JWKS when several services verify. Keys in KMS/Vault. Short lifetimes (minutes),
  because a JWT can't be revoked before `exp`. No secrets or PII in the payload. *(MacDonald)*
- **For a monolith, opaque server-side sessions beat JWTs:** simpler and instantly revocable.
  *(MacDonald; Feross L03)*

### APIs (OWASP API Security Top 10 2023)
**API1 BOLA** (most common; UUIDs slow guessing but are not the control) · **API2 Broken
authentication** (reset flows, JWT handling, no brute-force limits, GraphQL batching many
logins per request) · **API3 BOPLA** (data exposure + mass assignment, `"isAdmin": true`) ·
**API4 Resource consumption** (timeouts, upload/page size, batch ops, spend caps; deep JSON
crashes parsers) · **API5 BFLA** (user calls admin function; verb switching; admin of org A
acting in org B) · **API6 Sensitive business flows** (no code bug: scalping, booking hoarding,
coupon farming at machine speed) · **API7 SSRF** · **API8 Misconfiguration** (permissive CORS,
debug mode, stack traces, no TLS internally, leaky headers, cached private responses) ·
**API9 Inventory** (forgotten `/v1`, `/beta`, staging on prod data) · **API10 Unsafe
consumption** (third-party API and LLM output is untrusted; Experian 2021 trusted a partner
that exposed credit checks). *(Corey Ball; Paxton-Fear)*

### OWASP Top 10 2025 (web apps)
A01 Broken Access Control (still #1) · A02 Security Misconfiguration (up from #5) · A03 Software
Supply Chain Failures (new, wider than "outdated components") · A04 Cryptographic Failures ·
A05 Injection (down from #3) · A06 Insecure Design · A07 Authentication Failures · A08 Software
or Data Integrity Failures · A09 Security Logging & Alerting Failures · A10 Mishandling of
Exceptional Conditions (new: fuzz for Unicode, huge and malformed input). SSRF is now folded
into A01. Logic flaws (A06) need threat modelling: scanners and AI coders miss them (e.g. a
negative transfer; a real Coinbase logic bug allowed effectively unlimited money). *(Aikido; final list checked by reviewer)*

### Injection and SSRF
- **Parameterised queries are the primary fix; escaping is secondary.** Concatenated input lets
  an attacker go from one stray `'` to dumping `information_schema` and password hashes in a
  handful of requests. *(Pound)*
- **Second-order injection:** data from your own DB is not safe; parameterise every query.
  Least-privilege DB users, one DB per app, generic errors, capped rows. *(Pound)*
- **A server that fetches URLs is a confused deputy.** Priority: don't accept URLs (accept an
  ID); allow-list host, scheme (http/https only), port; isolate the fetcher with no
  credentials and no internal reach; require IMDSv2 and least-privilege instance roles;
  disable or re-validate redirects; one URL parser; never return raw fetched responses.
  Capital One lost ~100M records via SSRF → metadata credentials → S3. *(McKenzie/Aikido;
  Corey Ball API7)*
- **DNS rebinding:** a hostname can resolve to a public IP at check time and to `127.0.0.1`
  at fetch time. Resolve once, validate the IP (private, loopback, link-local incl. IPv6 and
  odd encodings), connect to that pinned IP, repeat per redirect. *(Reis; notes)*

### Secrets and supply chain
- **Model the whole lifecycle:** creation → storage → distribution → use → rotation → audit.
  Most leaks happen in hand-offs (email, wiki, properties file, git). Base64 is not
  encryption. *(Hardeep & Hughes/Barclays)*
- **One credential per service per environment,** owned by an application, short-lived or
  automatically rotated with a two-key overlap; solve "secret zero" with workload identity
  (cloud IAM role, service account), hold secrets in memory. *(Barclays)*
- **The supply chain includes your IDE and CI:** hijacked npm packages (debug/chalk, ~2.6B
  weekly downloads), typosquats (one study ran on ~17k domains), malicious VS Code/Cursor
  extensions stealing local secrets. Use SCA *plus* malware scanning, few dependencies,
  lockfiles as a minimal SBOM, publish only from CI. *(Aikido A03/A08; Feross L01; Crume)*
- **AI-written code is an untrusted contributor** that gets less review because there is so
  much of it. Review it like a stranger's PR; don't paste keys or proprietary code into tools
  whose data policy you haven't checked. *(Crume)*

### Payments and bookings
- **A timeout means "unknown", not "failed".** Retrying a charge whose response was lost moves
  money twice. *(Arpit Bhayani)*
- **Idempotency key per user intent, generated once** (when checkout loads, not per HTTP
  attempt), persisted before money moves, sent on every retry; enforced by a unique constraint
  or `INSERT … ON CONFLICT DO NOTHING`; states `pending → processing → succeeded/failed`;
  store the response; bind key to parameters (same key, different amount → 409/422); expire
  (~24h like Stripe); check status by ID before retrying; dedupe webhooks on event ID.
  *(Bhayani, with notes)*
- **Don't auto-retry what isn't idempotent.** For low-stakes actions, showing an error may be
  enough; for money you need idempotency *and* no blind retries. *(Bhayani, with note)*
- **PCI scope = anything that stores, processes or transmits card data, plus what connects to
  it.** Cheapest compliance: card data never touches your servers. Use the PSP's hosted page,
  redirect or hosted fields; store only tokens; never store CVV/SAD, even encrypted; segment;
  get vendors' AOCs. Card numbers in logs, error trackers or LLM prompts pull those systems
  into scope. Hosted page/redirect usually means SAQ A instead of SAQ D, but SAQ A now
  requires your site to be "not susceptible to script attacks" (section 7).
  *(Vaswani/ControlCase — a v3.2.1-era webinar; current standard is v4.0.1)*

---

## 3. Checklists

**A. AI-written code security review** (run on every AI diff touching I/O)
- [ ] Every handler taking an ID filters by the caller (BOLA); admin routes check role (BFLA).
- [ ] No `Model.update(req.body)` / `**request.json` into ORM; no whole-object `to_json()`.
- [ ] No f-string/concatenated SQL, shell (`shell=True`, `os.system`) or `eval`; identifiers
      allow-listed.
- [ ] No `jwt.decode` without verification or with header-chosen `algorithms`; `iss`/`aud`/`exp`
      checked; keys only from your configured JWKS (ignore `jku`/`x5u`).
- [ ] Password hashing is argon2id (or bcrypt with the 72-byte issue handled), via a library.
- [ ] Cookie flags set; no `res.cookie('userId', …)`; no tokens in localStorage or URLs.
- [ ] Login/reset: no early return (timing), identical errors and statuses, rate limited.
- [ ] No hard-coded keys "for testing"; no logging of request bodies/headers with secrets.
- [ ] Each new dependency exists, is spelled right, maintained and pinned (typosquat bait).
- [ ] Every `requests.get(url)`/`fetch(url)` on external input goes through the safe fetcher.
- [ ] No generic retry decorator (tenacity, axios-retry) wrapping a non-idempotent POST.
- [ ] Invariants written and tested (no negative amounts, server-side price, refund ≤ charge).
- [ ] OAuth code uses code + PKCE + `state`, never implicit.
- [ ] Payment forms do not post card fields to your own API.

**B. Payments / booking flow**
- [ ] Booking intent row persisted with idempotency key *before* calling PSP or supplier.
- [ ] Unique constraint on the key; key bound to amount/currency/user; response stored.
- [ ] Supplier reference (partner/client ref) sent on every book call; status checked by that
      reference before any retry.
- [ ] Timeouts = "unknown": check status, never blind retry; reconcile with PSP/supplier.
- [ ] Charge-ok/booking-failed gets a compensating refund via an explicit state machine.
- [ ] Price, currency, add-ons and commission recomputed server-side from the quote.
- [ ] Webhooks: HMAC verified on the raw body in constant time, deduped on event ID,
      transaction re-fetched from the PSP before fulfilling, handler idempotent.
- [ ] Card entry via PSP hosted page/redirect/hosted fields; only tokens stored; no PAN/CVV in
      logs, error trackers, analytics or prompts.
- [ ] Booking endpoints authorise per object; vouchers/itineraries not fetchable by ID alone.
- [ ] Abuse controls on holds, free-cancellation and searches that burn supplier quota (API6).
- [ ] Re-auth before payout or bank-detail changes; CSRF defence on every money endpoint.

**C. Browser and session**
- [ ] HTTPS + HSTS (preload only when all subdomains are HTTPS); cookie `HttpOnly; Secure;
      SameSite=Lax; Path=/` set explicitly, no `Domain=`, explicit expiry; server row deleted
      on logout and password change.
- [ ] CSRF token or Origin/`Sec-Fetch-Site` check on state changes; no mutation on GET.
- [ ] CSP (nonces/hashes), SRI on third-party scripts, `frame-ancestors`, `nosniff`.
- [ ] CORS exact-origin allow-list; never reflect any `Origin` with credentials;
      `postMessage` origin-checked.
- [ ] Sensitive apps and admin on their own subdomain.

**D. Page fetcher / LLM pipeline (SSRF + prompt injection)**
- [ ] Fetcher runs as an isolated worker: no secrets, no cloud role, egress allow-list or proxy,
      no route to internal ranges or `169.254.169.254` (IMDSv2 if on AWS).
- [ ] Scheme http/https only; ports 80/443; one URL parser; resolve → validate IP → connect to
      pinned IP; re-validate every redirect hop (cap ~3–5).
- [ ] Timeouts, max response size, content-type allow-list; raw responses never echoed to users.
- [ ] Never forward your cookies, tokens or supplier keys to fetched URLs.
- [ ] Fetched HTML is untrusted *text* for the model: delimit it, tell the model it is data,
      and never let it choose tools, URLs or recipients.
- [ ] Model output is untrusted (API10): schema-validate, check IDs/prices against your DB,
      escape on render, never feed into SQL/shell/`eval`.
- [ ] No single agent holds private data + untrusted content + an outbound channel without a
      human gate; LLM-generated SQL runs read-only with row limits.
- [ ] Per-user rate limits and spend caps on LLM calls (API4).

**E. Secrets**
- [ ] Nothing in git (scan *history*), wikis, chat, notebooks or frontend bundles; `.env`
      git-ignored; pre-commit secret scanner (gitleaks/trufflehog) and CI scanning.
- [ ] Separate sandbox and production keys; one key per service where the supplier allows.
- [ ] Platform secret store in deploys; workload identity over static keys where possible.
- [ ] Least scope (PSP restricted keys); webhook signing secrets treated as secrets.
- [ ] Rotation runbook per key tested before you need it; two-key overlap for zero downtime.
- [ ] Logs, traces and error trackers redact `Authorization`, cookies, keys, passwords, card
      fields; LLM prompts never contain keys.
- [ ] JWT/session signing keys in KMS/Vault or platform store, not copied to every host.

---

## 4. Failure modes to hunt for in reviews

(Beyond checklist 3A.)
- Auth framework present but not called on one endpoint (the usual BOLA).
- Hidden button, "only admins should call this" doc, or client-side `multifactor=true`
  treated as a control.
- Signed cookie or long-lived JWT as a session; logout that only clears the cookie; password
  change that keeps old sessions alive.
- f-string SQL over scraped or supplier text (a place name with `'` breaks or injects).
- `requests.get(user_url)` hiding in a "preview", "import from link" or PDF feature.
- A new UUID minted per retry; old `/v1` or staging live on production data.
- Secrets in base64 or env files copied everywhere; logs nobody alerts on.

---

## 5. Where the teachers disagree, and how to resolve it

- **JWT vs sessions.** MacDonald likes JWTs when many services verify without a network
  call; Feross says don't put authority in unrevocable tokens. *Resolve:* monolith → opaque
  sessions; distributed verifiers → short-lived asymmetric JWTs + revocable refresh tokens.
- **Idempotency everywhere?** Bhayani says skip it where you can just show an error. *Resolve:*
  fine for likes and low-stakes actions; never for money or bookings, where an explicit user
  retry can still double-charge.
- **Generic auth errors vs usability.** Feross: generic everywhere, or friendly messages plus
  tight rate limits. *Resolve:* money apps (Tereka) go generic; low-sensitivity apps may be
  friendly if enumeration can't scale.
- **CAPTCHAs.** Aikido lists them; Feross shows they are cheap to beat. *Resolve:* rate
  limits, risk-based step-up MFA and bot/timing signals first; CAPTCHA as a speed bump.
- **One vault vs platform stores.** Barclays runs one federated vault. *Resolve:* a solo dev
  uses the hosting platform's secret store; the principles (per-service, short-lived, audited)
  matter more than the product.
- **Severity of API6.** Paxton-Fear frames it as broad business logic across integrations;
  OWASP's definition is narrower (abuse of a legitimate flow at scale). Use the narrow one for
  classification, the broad one for threat modelling.

---

## 6. Applying this to Sam's work

- **Trameter bookings (Viator, RateHawk):** BOLA on `/bookings/{id}` and vouchers is the #1
  risk; idempotency key + supplier reference + status check before retry; saga for
  charge-ok/book-failed; recompute price server-side; hosted payment fields so Trameter stays
  out of full PCI scope.
- **Drive Button / LLM page fetcher:** textbook SSRF and prompt-injection surface. Isolated,
  credential-free fetcher with pinned-IP checks; model output validated against your DB. The
  widget embedded in blogs runs in someone else's page: serve booking UI from your own origin
  in an iframe, `postMessage` with origin checks, confirm payment in a top-level window or
  popup, `frame-ancestors` allow-listing partners, SRI-pin the published script.
- **Data pipelines (GeoNames, OSM, supplier dumps):** bind parameters in DuckDB/Postgres/SQLite
  and pandas `read_sql`; allow-list identifiers; fuzz with odd Unicode and huge rows (A10).
- **Tereka (money planner):** managed OIDC login, MFA pushed hard, generic errors, re-auth for
  email/password changes, no card data at all; multi-tenant households need cross-household
  BFLA tests.
- **Riselist:** "connect TikTok" is delegated OAuth: minimal scopes, refresh tokens encrypted,
  revocation handled; dedupe Paystack/Flutterwave/Stripe webhooks.
- **Supplier keys and the laptop:** keys server-side only, sandbox/prod split, rotation known
  in advance, never in notebooks, prompts or either GitHub account; few editor extensions.

---

## 7. Reviewer corrections and "not covered by the sources, but know this"

*(Added by an independent application-security review, November 2026. Not from the talks.)*

**Corrections applied to the source material**
- **Passwords (NIST SP 800-63B-4, final 2025):** password-only login needs **15+** characters;
  8+ only when the password is one factor of MFA. Allow ≥64, any Unicode, no composition
  rules or forced rotation, blocklist check, never truncate. Feross's "8 minimum" is rev 3.
- **bcrypt pre-hash:** raw `bcrypt(sha256(pw))` can hit NUL-byte truncation and lets leaked
  unsalted SHA-256 hashes be "shucked". Prefer **argon2id** (OWASP baseline m=19 MiB, t=2,
  p=1); if stuck on bcrypt, pre-hash with HMAC-SHA256 under a pepper, base64-encoded.
- **OWASP Top 10 2025 (final):** A09 is "Security Logging & Alerting Failures"; SSRF is no
  longer its own category and sits inside A01.
- **PCI DSS v4.0.1:** v4.0's future-dated requirements became mandatory on 31 March 2025.
  SAQ A (January 2025 revision) dropped 6.4.3/11.6.1 but added an eligibility criterion that
  your site is "not susceptible to attacks from scripts" affecting the payment flow. With
  embedded iframe/hosted fields you still own the parent page's scripts (CSP, SRI, few tags);
  a full redirect is the simplest scope. Confirm your SAQ with the PSP/acquirer.
- **OAuth:** RFC 9700 (OAuth 2.0 Security BCP, Jan 2025): PKCE for *all* clients, no implicit
  or password grant, exact redirect URIs, rotating or sender-constrained refresh tokens for
  public clients. Check OAuth 2.1's status before citing it as an RFC.
- **JWT (RFC 8725):** ignore attacker-supplied `jku`/`x5u`; `kid` only selects from *your*
  JWKS; pin key type with the algorithm.
- **Browser details:** SameSite defaults differ by browser, so set it explicitly. CORS: the
  real bug is reflecting any `Origin` with credentials, not `*`. `X-XSS-Protection`: the
  auditor is gone; send `0` or omit it (not a weakness signal, contrary to the course). HSTS
  `preload` is hard to undo; add it only when every subdomain is HTTPS.
- **Numbers and retries:** the "~640x" cost curve rests on thin data (direction, not a
  figure). Where retries are safe, use capped exponential backoff with jitter at one layer.

**Not covered by the sources, but know this**
- **LLM/agent risks:** OWASP Top 10 for LLM Applications 2025: LLM01 Prompt Injection,
  LLM02 Sensitive Information Disclosure, LLM03 Supply Chain, LLM04 Data and Model Poisoning,
  LLM05 Improper Output Handling, LLM06 Excessive Agency, LLM07 System Prompt Leakage,
  LLM08 Vector and Embedding Weaknesses, LLM09 Misinformation, LLM10 Unbounded Consumption.
  Prompt injection has no complete fix: limit what an injected model *can do* (least-privilege
  tools, read vs write split, human approval for money/outbound actions, output validation,
  no secrets in prompts, egress allow-lists, no auto-rendered model-chosen URLs or images).
  Treat third-party MCP servers and their tool descriptions as untrusted code; pin them.
- **Passkeys (WebAuthn/FIDO2):** phishing-resistant, origin-bound, nothing reusable on the
  server to steal. Offer them as the preferred login and step-up factor; SMS OTP is the weakest
  MFA (SIM swap), TOTP is the middle ground. A managed IdP gives you passkeys for free.
- **Dependency hygiene in Python:** `uv lock` (or pip-tools with `--generate-hashes`) to pin
  exact versions and hashes; `uv sync --locked` in CI; `pip-audit` (or OSV-Scanner) in CI plus
  Dependabot/Renovate; add new packages only after checking name, maintainer and age;
  GitHub Actions pinned to commit SHAs; generate an SBOM (CycloneDX/Syft) for deployables.
- **Rate-limit design:** token bucket or sliding window (Redis or gateway) keyed per user,
  per IP and per API key; tighter buckets on login, reset, OTP and paid calls; `429` +
  `Retry-After`; take client IP only from a trusted `X-Forwarded-For` hop; global circuit
  breaker and spend alarms on LLM and supplier APIs.
- **Webhook verification:** compute the HMAC over the **raw** request body (before JSON
  parsing) with the provider's secret, compare in constant time (`hmac.compare_digest`),
  reject stale timestamps where the provider signs one (e.g. Stripe, ~5 min), then dedupe on
  event ID and re-fetch the transaction from the provider's API before fulfilling (Paystack
  signs with HMAC-SHA512 in `x-paystack-signature`; check each PSP's docs). Return 2xx fast
  and process async.
- **Logging without PII or secrets:** structured logs with an allow-list of fields, redaction
  middleware for `Authorization`, cookies, passwords, tokens, card and bank fields;
  pseudonymise user IDs/emails in analytics; scrub error trackers (Sentry `before_send`);
  short retention; separate security audit log. LLM traces are logs too.
- **NDPA/GDPR basics for a Nigerian product:** the Nigeria Data Protection Act 2023 is the
  governing law (the NDPR 2019 stopped applying once the NDPC's General Application and
  Implementation Directive, GAID 2025, took effect in September 2025). Practical minimum:
  lawful basis and a clear privacy notice; collect only what you need; support access,
  correction, deletion and portability requests; DPIA before high-risk processing; notify the
  NDPC within **72 hours** of a personal-data breach (and users promptly if high risk); check
  whether you are a "data controller of major importance" that must register and file audit
  returns; cross-border transfers need an adequacy basis or safeguards. Serving EU/UK users
  brings in GDPR (same 72-hour breach clock, records of processing, processor contracts). This
  is orientation, not legal advice.
- **Incident basics:** a one-page runbook to revoke all sessions, rotate every key and
  disable a supplier integration in minutes.

---

## 8. Where to look in `references/`

| Need | File (see INDEX.md) |
|---|---|
| Authentication and password storage (Feross L13), web security mindset (L01), OWASP Top 10 2025 (Aikido), JWT pitfalls (MacDonald/CNCF), PCI DSS (Vaswani/ControlCase), idempotency keys (Arpit Bhayani) | `sec-1-*` |
| Sessions and cookies (Feross L03), in-flight modification, CSP origins, DNS rebinding (Charlie Reis), OWASP API Top 10 attacker view (Paxton-Fear), secrets lifecycle (Barclays/HashiConf), SOP/cookies/HSTS basics (Rehberger), DevSecOps/SBOM/AI code (Crume/IBM) | `sec-2-*` |
| CSRF and SOP (Feross L04), OWASP API Top 10 course (Corey Ball), OAuth 2.0 and OIDC (Barbettini/OktaDev), SQL injection walk-through (Pound/Computerphile), SSRF and Capital One (McKenzie/Aikido) | `sec-3-*` |
