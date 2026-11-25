# sec-3 — cross-cutting principles

1. **Never trust the client — or anything that arrives over the wire.** Browser requests, API bodies, IDs, URLs, third-party API responses and even values already in your own DB (second-order injection) are attacker-controllable. (Feross CS253; Corey Ball API10; Mike Pound/Computerphile)
2. **Keep data from becoming code.** SQLi, command injection, XSS and SSRF share one root: user data reaches an interpreter or a fetcher. Fix structurally: parameterised queries, allow-listed identifiers, never concatenate. (Computerphile; Corey Ball injection; Aikido SSRF)
3. **Authorise every object, every function, every field, on every request.** BOLA (whose object?), BFLA (which action/role?), BOPLA (which properties may be read/written?) are the most common real API breaches and scanners miss them; use one central deny-by-default authz module plus authz tests in CI. (Corey Ball API1/3/5)
4. **Return the minimum and accept the minimum.** Explicit response DTOs, no `to_json()` of whole models, allow-listed writable fields, no auto-binding — stops excessive exposure and mass assignment (`isAdmin: true`). (Corey Ball API3)
5. **Ambient authority is the root of CSRF.** Browsers attach cookies to any request to your domain whatever page triggered it; SOP blocks reading responses, not sending forms. Session cookies: `Secure; HttpOnly; SameSite=Lax; Path=/`, plus CSRF/Origin checks on state-changing endpoints; never mutate on GET. (Feross CS253)
6. **Know the boundary you're actually relying on.** Origin = scheme+host+port; cookies scope by domain; SameSite scopes by registrable site — so other ports and sibling subdomains are "same-site". Cookie `Path` is not a boundary. Put login on its own subdomain. (Feross CS253)
7. **Cross-origin communication is opt-in and must be origin-checked.** Use `postMessage` with an explicit target origin and verify `event.origin`; use deliberate CORS policies; never `document.domain`. (Feross CS253; Corey Ball API8)
8. **Don't share passwords — delegate with scoped, short-lived tokens.** OAuth 2.0 = delegated authorisation with minimal scopes; OIDC ID token = authentication. An access token or API key is not proof of user identity. (Nate Barbettini/OktaDev; Corey Ball API2)
9. **Secrets and bearer tokens live on the back channel only.** Client secrets, access/refresh tokens and supplier API keys never go to the browser, front-end bundle or logs; the auth-code exchange exists precisely so the browser only sees a useless code. Use code + PKCE (implicit and password grants are deprecated). (OktaDev; Corey Ball/Experian example)
10. **Verify tokens fully.** JWTs: reject `alg: none`, check signature, `exp`, `iss`, `aud`; remember the payload is readable; use strong signing keys; resource servers enforce scope per action. (OktaDev; Corey Ball API2)
11. **Treat recovery endpoints as login endpoints.** Brute-force limits stricter than normal rate limits, lockout, re-auth for email/MFA changes; a 6-digit reset code is only 10^6 guesses. Prefer standard/hosted identity over homegrown auth. (Corey Ball API2; OktaDev)
12. **Put a limit on everything that costs money or CPU.** Rate, payload size, page size, batch ops, timeouts, memory, and third-party spend caps; protect business flows (checkout, holds, sign-ups) from bot-speed abuse with human/timing signals. (Corey Ball API4/API6)
13. **A server that fetches URLs is a confused deputy.** Avoid user-supplied URLs; otherwise allow-list host/scheme/port, isolate the fetcher with no credentials and no internal network reach, disable/re-validate redirects, block metadata IPs and use IMDSv2 — Capital One lost ~100M records this way. (Aikido SSRF; Corey Ball API7)
14. **Over-privilege turns small bugs into breaches.** A do-everything service, an IAM role that can read all S3, a DB user that can read `information_schema` and every co-hosted DB — shrink each so a hijack gains little. (Aikido; Computerphile)
15. **Your third-party APIs are an attack vector into you.** Validate/sanitise supplier responses like user input, enforce TLS, don't follow redirects blindly, assess the provider's security, and assume partner keys will leak. (Corey Ball API10 + Experian case)
16. **Attackers learn from every oracle.** Verbose SQL errors, stack traces, `X-Powered-By`, response-time differences (25 ms vs 510 ms) and status codes enumerate your system — uniform, schema-enforced errors. (Corey Ball API8; Computerphile timing/blind SQLi)
17. **What you don't inventory you don't protect.** Old `/v1/`, `/beta/`, staging hosts wired to prod data are the soft entry point; inventory hosts, versions and integrations, retire old versions, no prod data in non-prod. (Corey Ball API9)
18. **Threat-model business logic; don't rely on "please don't".** Docs saying "only admins should call X" or a client-side `multifactor=true` flag are trust, not controls; assume clients bypass the UI and edit every request. (Corey Ball, business logic)
19. **Log what attackers do and protect the logs.** Failed logins, denials, validation errors, with integrity and alerts — without them breaches go unnoticed. (Corey Ball, logging & monitoring)

## Feross Aboukhadijeh (Stanford CS253) — Web Security Lecture 04: Cross-Site Request Forgery, Same Origin Policy (https://www.youtube.com/watch?v=0-q69vAYSwo)
**Core idea: the browser attaches cookies "ambiently" to any request to a site, so any other site can make the user's browser send authenticated requests; the Same Origin Policy stops other sites *reading* responses, but not legacy *sending* (forms, images) — so state-changing endpoints must defend themselves.**

**Mental models / first principles**
- Cookies carry "ambient authority": every request to the cookie's domain gets the user's session attached automatically, regardless of which page triggered it. Convenient, and the root cause of CSRF.
- Same Origin Policy (SOP) is "the fundamental security model of the web": pages from different origins must not interfere with each other. Think of the browser as an OS kernel and each origin as a process with isolated memory. Security depends on the browser implementing this correctly.
- Origin = (scheme, host, port). `www.example.com` vs `example.com` are different origins; `http` vs `https` differ; `:80` vs `:8080` differ; an explicit `:80` equals the http default.
- Cookies and SOP were designed separately (cookies ~1994, SOP ~a year later), so their rules disagree: cookies scope by domain (and path), not by full origin. Port and scheme don't isolate cookies.

**Cookie scoping mechanisms**
- A cookie set for a parent domain is readable by more-specific subdomains (`stanford.edu` cookie visible to `cs253.stanford.edu`), not the reverse; sibling subdomains can't read each other's. This is why Stanford runs login on its own subdomain (`login.stanford.edu`) — student-club subdomains can't read session cookies.
- `Path` is NOT a security boundary: a same-origin page can iframe the other path and read its `document.cookie`. Advice: always set `Path=/` explicitly so you don't fool yourself (default is the current path, which surprises people).
- Recommended session cookie: `Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=~30 days` (refresh the expiry on each visit). `HttpOnly` hides the cookie from page JavaScript, limiting damage if XSS gets code running.
- Clearing a cookie: re-send it with a past expiry AND identical attributes (domain/path/etc.), or the browser treats it as a different cookie and leaves the old one.

**CSRF mechanism**
- Attacker page embeds `<img src="https://bank.example/withdraw?from=bob&to=mallory&amount=1000">`; the browser sends it with Bob's cookies; the server sees a valid session and acts. The attacker never needs to read the response — the side effect is the damage.
- Switching to POST does not fix it: the demo hides an auto-submitting `<form action="http://localhost:4000/transfer">` inside a hidden iframe on a "cat pictures" page; every page load silently transferred money. Cross-site form posts are an SOP exception kept for backwards compatibility.
- Worse against admins: forged requests to an admin panel can create admin users or even run code on the server.
- Attacker needs to know the endpoint and that the victim is logged in — easily combined with phishing/targeting.
- GET must never change state (otherwise images/links can trigger it), but GET-vs-POST is not a defence by itself.

**SameSite attribute (defence)**
- `None`: old default, always send. `Lax`: don't send on cross-site sub-resource requests (images, iframes, form POSTs from another site) but do send on top-level navigation (clicking a link). `Strict`: never send if the navigation originated from another site, even a plain link — users arrive logged out; maybe for finance, usually too aggressive. Lax is the right default for most sites.
- What breaks with Lax: embedded third-party widgets that need login (e.g. Facebook comment iframe), ad tracking.
- Google's proposal (since shipped in Chrome): cookies are Lax by default; `SameSite=None` requires `Secure`.
- Gotcha from the live demo: SameSite is about *site* (registrable domain), not origin — `localhost:9999` attacking `localhost:4000` is "same-site", so the cookie was still sent. Only when the attacker was on a different hostname did the browser send `Sec-Fetch-Site: cross-site` and drop the cookie. Lesson: two apps on different ports or sibling subdomains of the same domain are same-site to each other — SameSite won't protect them from each other.
- `Sec-Fetch-Site` / `Sec-Fetch-*` request headers tell the server how and from where a request was triggered; usable as a server-side check.
- (note: the lecture defers full CSRF tokens to a later lecture; SameSite is described as solving it "for the most part", not completely — keep CSRF tokens or Origin/Sec-Fetch checks on state-changing endpoints for defence in depth.)

**What SOP allows vs blocks**
- Allowed cross-origin: linking; embedding iframes, images, scripts (CDN use case); submitting forms; navigating a child iframe to a new URL (a phishing risk — the parent can swap what's shown in the frame).
- Blocked: reaching into a cross-origin iframe's DOM (`iframe.contentDocument` is null); reading a cross-origin `fetch()` response (CORS error). Rationale: fetching Gmail from an arbitrary page would send your cookies and hand your inbox to that page. A server-side `curl` isn't dangerous because it has no victim cookies.
- `fetch` is a newer API, so it enforces SOP strictly; servers can opt in to cross-origin reads (CORS) when building public APIs. Secure by default; relaxing is explicit.
- SOP is sometimes too narrow (login and access subdomains want to cooperate) and sometimes too broad (different teams' sites on the same origin, e.g. multiple courses under one host, fully trust each other).

**Relaxing SOP — bad and good ways**
- `document.domain = "stanford.edu"`: both pages must opt in, scheme/port can't change, but then *any* subdomain can also join the party. Never use it; treat sightings as a red flag (note: modern browsers have deprecated/disabled it).
- Fragment-identifier hacks (parent sets iframe `#hash`, child polls every 100 ms) — historical curiosity.
- `postMessage(data, targetOrigin)`: structured clone of objects (handles cycles; no functions/DOM nodes), transferable objects for zero-copy handoff of large buffers. Always pass an explicit target origin when sending, and in the `message` handler check `event.origin` against an allow-list before acting — "really bad things happen" otherwise.

**Rules of thumb**
- Never trust anything from the client; ambient authority is useful but risky.
- Session cookies: `Secure; HttpOnly; SameSite=Lax; Path=/`, bounded Max-Age.
- Never mutate state on GET. Add CSRF protection (SameSite + token or Origin/Sec-Fetch-Site check) on every state-changing endpoint, especially money movement and admin actions.
- Put auth/login on a dedicated subdomain; don't host untrusted or other-team content on the same origin or same site as sensitive apps.
- Validate `event.origin` on every `postMessage` receiver.

**Relevance to Sam**
- Booking/payment and Tereka endpoints with cookie sessions: a cross-site form can trigger bookings, transfers or settings changes unless SameSite + CSRF checks are in place. Payment endpoints also need transactions/idempotency (the teacher's naive debit/credit admits this).
- Staging/admin tools on `localhost` ports or sibling subdomains are same-site to each other — don't rely on SameSite there.
- Embeddable widgets (e.g. the Drive Button injected into blog pages) live in a cross-origin context: talk to the parent via `postMessage` with strict origin checks; expect third-party cookies inside iframes to be dropped under Lax.
- AI-written Express code often omits cookie flags and origin checks — audit for them.

## Corey Ball (APIsec University, via freeCodeCamp) — OWASP API Security Top 10 Course – Secure Your Web Apps (https://www.youtube.com/watch?v=YYe0FdfdgDU)
**Core idea: APIs hand attackers a direct, structured path to data and functions, bypassing decades of perimeter defences; most real breaches are authorisation and trust failures that scanners cannot see, so the defence is per-request authorisation, minimal data in/out, limits on everything, and zero trust in anyone's input — including third-party APIs.**

**Mental models / first principles**
- Why APIs are the new front door: an attacker no longer needs the classic kill chain (breach firewall, pivot, exfiltrate); an insecure API hands over the data directly. Traditional web scanners and network monitoring were not built for API flaws.
- The 2023 list was compiled from bug-bounty write-ups and public breaches (no community data was submitted) — biased toward what bounties reward, but real.
- Risk = likelihood x impact; OWASP scores only exploitability/prevalence/detectability/technical impact. Business impact is yours to assess — the list is a guide, not your risk analysis.
- 2019 -> 2023: Injection and Logging/Monitoring dropped off (still real; WAFs reduced notable injection, and logging gaps don't show up in breach reports). BOLA, BFLA, Misconfiguration stayed. New: SSRF, Unsafe Consumption, BOPLA (= Excessive Data Exposure + Mass Assignment), Unrestricted Resource Consumption, Sensitive Business Flows.
- "Complexity is the worst enemy of security" (Schneier) — stateless REST pushes auth into tokens, so registration, token issuance, reset and MFA each become attack surface.

**API1 BOLA (broken object-level authorisation)** — most common and impactful.
- Attacker swaps an object ID (path, query, header, body) — e.g. `/api/v3/users?id=2727` -> `2728` returns another patient's record. IDs may be integers, UUIDs or strings; resources may be keyed by user, group or both.
- Scanners rarely catch it; developers have an auth framework but forget to call it on one endpoint.
- Fix: check, in every function that takes a client-supplied ID to fetch a record, that the logged-in principal may perform this action on this record. Random IDs (GUIDs) slow guessing but are not the control. Write authorisation tests (user A requests B's object -> must fail) and block deploys that fail them.

**API2 Broken Authentication**
- Weak password policy, no brute-force limits, changing password/email without re-confirming the password, tokens/passwords in URLs, GraphQL allowing many login attempts in one request (batching), credential stuffing from breach dumps, predictable tokens.
- JWT pitfalls: accepting unsigned tokens (`alg: none`), not checking expiry, sensitive data in the (merely base64-encoded) payload, weak signing secrets.
- Reset-code maths: a 6-digit code needs at most 1,000,000 guesses, 4-digit only 10,000 — trivial without rate limits.
- Fixes: know every auth flow (mobile, web, deep links, reset); don't invent auth, token generation or password storage — use standards; treat forgot-password/recovery endpoints exactly like login (rate limit, lockout); stricter anti-brute-force limits on auth endpoints than general rate limits; re-authenticate for sensitive changes (email, MFA phone); MFA; weak-password checks.
- "OAuth is not authentication, and neither are API keys." API keys identify a *client application*, not a user.

**API3 BOPLA (broken object property-level authorisation)**
- Excessive data exposure: endpoint returns the whole object and expects the front-end to filter (asked for a name, got DOB, email, phone and friends). Scanners can't judge it; it bypasses every other control because it's the "intended" API.
- Mass assignment: framework auto-binds request JSON onto the model, so adding `"isAdmin": true` (or a balance field) at signup/update escalates privilege. Attackers find hidden fields via docs, responses, fuzzing, or side effects.
- Fixes: never `to_json()` the whole model — cherry-pick response fields; avoid auto-binding client input to internal objects; allow-list writable properties per endpoint; schema-validate responses too; return the bare minimum the feature needs.

**API4 Unrestricted Resource Consumption** (was "Lack of Resources & Rate Limiting")
- Every request has CPU, memory and money cost; missing limits mean DoS or a surprise bill (auto-scaling infra, paid third-party calls like SMS/email/LLM).
- Limits to check: execution timeouts, max memory, file descriptors, processes, upload size, operations per request (GraphQL batching), page size/records per response, and spending caps on third-party providers.
- Fixes: container limits (Docker CPU/mem/restarts); per-client rate limits with a response telling the limit and reset time; server-side validation of `limit`/`page_size` params; max lengths for strings and max array sizes on all inputs.

**API5 BFLA (broken function-level authorisation)**
- BOLA = read someone else's account; BFLA = do something you're not allowed (transfer funds from others, admin actions). Lateral (another group's functions) or vertical (admin).
- Discovery: guess admin paths (`/admin/account/{id}` beside `/{id}/account/balance`), switch HTTP verbs (GET->PUT/DELETE), add yourself to groups.
- Fixes: one consistent, central authorisation module called by every business function; deny by default with explicit grants per role; admin controllers inherit from a base that enforces role checks; review endpoints against the role hierarchy.

**API6 Unrestricted Access to Sensitive Business Flows** (new)
- No code bug — the flow itself is abused at machine speed: scalpers draining stock (PS5 bots), spam, booking/reservation hoarding, gaming internal economies.
- Mitigate in two layers: business names the flows that hurt if overused; engineering picks controls — device fingerprinting, CAPTCHA/human detection, non-human timing patterns (add-to-cart and checkout under one second), blocking Tor exits/known proxies, and securing machine-to-machine APIs, which often lack these protections.

**API7 SSRF**
- Server fetches a URL the client controls -> internal service enumeration, firewall bypass, credential theft, RCE, or using you as a proxy. In-band (response returned to attacker) vs blind (attacker needs their own server to observe the callback). Example: a "check stock" URL parameter swapped for a localhost admin path returned a password.
- Fixes: isolate the fetcher in its own network segment; allow-list origins, schemes, ports and media types; disable redirects; use one well-tested URL parser (parser disagreements cause bypasses); never return raw fetched responses to the client.

**API8 Security Misconfiguration** (catch-all, the only category scanners like Burp, ZAP, Nessus, Nikto detect well)
- Unpatched components, default accounts, unnecessary HTTP verbs, missing TLS (even internal/partner APIs — Wireshark reads cleartext), verbose errors/stack traces, missing input sanitisation (upload a script into a web directory -> shell), leaky headers.
- Headers as intel: `X-Powered-By` reveals stack+version for exploit lookup; `X-XSS-Protection: 0` advertises missing control; `X-Response-Time` as a side channel — 25 ms for nonexistent records vs 510 ms for real ones lets an attacker enumerate valid accounts.
- Fixes: repeatable hardening and fast redeploy; continuous config review across orchestration files and cloud services; TLS everywhere; allow only needed verbs; proper CORS for browser clients; security headers; restrict content types; uniform request handling across proxies (avoid request desync/smuggling); schema-enforce error responses too.

**API9 Improper Inventory Management**
- Forgotten `/v1/`, `/beta/`, `/test/`, `/uat/`, `api.test.target.com`, `/api/partner`, version in headers (`Accept: version=2.0`) or params — old or dev versions lack the newer fixes and are often wired to the same production database.
- Found via stale docs, changelogs, repo history, Google dorking, DNS enumeration, fuzzing.
- Fixes: inventory every host (env, who can reach it, version) and every integrated service with what data flows and its sensitivity; auto-generate docs from open standards in CI; protect all exposed versions, not just current; no production data in non-prod (or protect it equally); when a new version fixes security, backport or force migration.

**API10 Unsafe Consumption of APIs** — the consumer-side risk.
- Treat third-party API data like user input: little to no trust. If the provider is compromised, your integration becomes the attacker's path in (injection, data exposure, DoS).
- Fixes: assess the provider's security posture; TLS for all calls; validate and sanitise everything received; allow-list redirect destinations and never follow redirects blindly.

**Beyond the Top 10**
- Injection (SQL, NoSQL, LDAP, OS command, XML, ORM): signs include DB error text in responses (e.g. "error in your SQL syntax"), odd status codes, timing. Fixes: keep data separate from commands; parameterised APIs; one maintained validation library; validate data from integrated systems too; escape for the specific interpreter; strict types/patterns per parameter; cap returned rows to limit blast radius.
- Logging & monitoring: log failed auth, access denials, validation errors with enough detail to identify actors; treat logs as sensitive and protect their integrity; centralise (SIEM), dashboards and alerts.
- Business logic flaws: features used against you because the design assumes good behaviour. Red flags in docs: "only use X for Y", "only admins should call Z". Example: a login request with `multifactor=true` that the client can flip to false. Experian 2021: a partner exposed its credit-check API calls via its web app; anyone could get credit scores by name+address — Experian trusted the partner. Stolen keys turn a trusted consumer into a "wolf in sheep's clothing". Fixes: threat-model flows at design time, remove trust relationships, assume clients bypass the browser and edit any request, external testing/bug bounty.

**Rules of thumb / checklist**
- Every endpoint: who is the caller, may they act on *this* object, may they call *this* function, may they read/write *these* fields?
- Deny by default; central authz module; authz tests in CI.
- Explicit response DTOs and writable-field allow-lists; no auto-binding.
- Limits on rate, size, page size, timeouts, batch ops, and third-party spend.
- Recovery endpoints = login endpoints.
- Inventory and retire old versions; no prod data in staging.
- Third-party responses = untrusted input; don't follow redirects blindly.

**Relevance to Sam**
- Trameter booking APIs: BOLA on `/bookings/{id}` is the #1 risk — always scope queries by the authenticated user; BFLA on admin/supplier endpoints.
- Viator/RateHawk/Coras responses are API10 input: validate schemas, sanitise strings before rendering or storing, cap sizes, enforce TLS, don't follow redirects; plan for a supplier being compromised.
- Supplier API keys authenticate Trameter as a client, never a user — and should never reach the browser (cf. Experian partner leak).
- LLM pipeline fetching blog/web pages is textbook SSRF territory: allow-list, isolated fetcher, no redirects, never echo raw responses.
- Paid calls (LLM tokens, supplier search quotas, SMS) need per-user rate limits and spending caps (API4); booking holds/inventory need business-flow abuse controls (API6).
- AI-written code commonly returns whole ORM objects and binds request bodies directly — check every handler for BOPLA.
- Tereka/Riselist: password reset codes need rate limiting; re-auth for email changes; validate JWTs (alg, expiry, secret strength).

## Nate Barbettini (OktaDev) — OAuth 2.0 and OpenID Connect (in plain English) (https://www.youtube.com/watch?v=996OiexHze0)
**Core idea: OAuth 2.0 is for delegated *authorisation* (letting an app do a scoped thing on your behalf without your password); OpenID Connect is a thin layer on top that adds *authentication* (an ID token saying who logged in). The authorisation-code flow splits work between the untrusted browser (front channel) and the trusted server (back channel) so secrets and tokens never touch the browser.**

**Mental models / first principles**
- Baseline "forms auth": username/password form, server checks a (hashed) password in its DB, sets a session cookie. Works, but you own every security and compliance burden forever (hashing changes, HIPAA, GDPR). Moving auth to a dedicated identity provider lets app and auth evolve separately.
- The problem OAuth solved: delegated authorisation. Pre-OAuth, Yelp asked users for their actual Gmail password to scrape contacts — handing over the "keys to the kingdom" (email resets everything else). OAuth replaces password sharing with a scoped, revocable token. (Speaker's 2018 aside that personal-finance aggregators still took bank passwords — note: open-banking/OAuth APIs have since spread, but screen-scraping persists in places.)
- Why OAuth docs are confusing: dense jargon plus spec ambiguity plus people misusing OAuth for login, so online advice contradicts itself.
- Front channel vs back channel: the browser is "trusted only as far as you can throw it" — URLs, view-source, devtools, extensions, shoulder-surfers can leak anything in it. Server-to-server HTTPS calls are the back channel and can hold secrets.

**Vocabulary**
- Resource owner = the user. Client = the app wanting access (Yelp). Authorization server = where the user logs in and consents (accounts.google.com). Resource server = the API holding the data (Contacts API); may or may not be the same system. Authorization grant = proof the user consented. Redirect URI / callback = where the auth server sends the browser back. Access token = the key the client presents to the resource server. Scopes = granular permissions (`contacts.read` vs `contacts.write`), space-separated, which drive the consent screen and limit the token.
- Client registration gives a client ID (public, sent via browser) and client secret (server-only; never share it, not even with a debugging tool).

**Authorization code flow mechanism**
1. User clicks "Connect with Google"; browser is redirected to the auth server with `client_id`, `redirect_uri`, `response_type=code`, `scope`, and `state` (an opaque value echoed back — note: use it as an unguessable per-request value to bind the callback to the user's session and stop login CSRF).
2. User logs in on the provider's own domain (visible in the address bar), sees a consent screen listing scopes, clicks yes (or no -> error redirect).
3. Auth server redirects to the callback with `?code=...` — visible in the URL/front channel.
4. Client's backend POSTs code + client_id + client_secret to the token endpoint (back channel) and receives an access token with an expiry (minutes to hours).
5. Client calls the resource server with `Authorization: Bearer <token>`. The API must validate: not forged, not expired, and scope covers *this* action (a read-only contacts token must not delete contacts).
- Why the two-step code exchange: a stolen code (malicious extension, logged URL) is useless without the client secret, and the access token never passes through the browser. Access tokens are bearer credentials — whoever holds one can use it within its scope — so keep them server-side.

**Other grant types (as presented, with corrections)**
- Implicit (`response_type=token`): token returned directly in the browser, for SPAs with no backend. Less secure — token exposed to the browser; requires no XSS. (note: this is outdated. Current OAuth best practice / OAuth 2.1 deprecates implicit; SPAs should use authorization code + PKCE, ideally with a backend-for-frontend holding tokens.)
- Resource owner password credentials: app collects the user's password and swaps it for a token; for legacy apps only. (note: deprecated in OAuth 2.1 — it reintroduces password sharing.)
- Client credentials: back-channel only, machine-to-machine, no user involved — the right fit for service-to-service calls.
- Native mobile: authorization code + PKCE (Proof Key for Code Exchange), using the system browser tab (Custom Tabs / SFSafariViewController), via the AppAuth library; store tokens in the OS keychain.

**OpenID Connect**
- OAuth has no standard way to learn *who* the user is, so Facebook/Google/etc. each bolted on custom, non-interoperable hacks to build "Log in with..." buttons. OIDC standardises it: request scope `openid` (that's the only on-the-wire difference) and receive an **ID token** alongside the access token, plus a standard **/userinfo** endpoint callable with the access token for more profile data.
- ID token is a JWT: header, payload (claims: user id/`sub`, email, issued-at, expiry, issuer, audience), and signature. The client verifies the signature cryptographically to confirm the auth server issued it and nobody edited it — no round trip needed. (note: also verify `iss`, `aud` = your client_id, `exp`, and `nonce`; base64 is encoding, not encryption — claims are readable by anyone holding the token.)
- Use OAuth for authorisation/permissions; OIDC for authentication (login, SSO, mobile). OIDC doesn't replace OAuth — it replaces *misusing* OAuth for login. SAML remains common for enterprise SSO; an identity provider can bridge SAML so your app speaks only OIDC.

**Typical architectures**
- Server-rendered web app: OIDC authorization code flow; backend keeps access + ID tokens; browser just gets a normal session cookie.
- Mobile: code + PKCE, tokens in keychain.
- SPA + API: (modern) code + PKCE or a BFF; guard against XSS because any token in JS is stealable.

**Rules of thumb / gotchas**
- Never ask users for third-party passwords; use OAuth scopes.
- Request the minimum scopes; consent screens should be explicit (Facebook had to split "post to your wall" out after abuse complaints).
- Client secret and access tokens live only on the server; client ID is public.
- Resource servers must check token validity, expiry and scope on every request.
- An access token is not proof of identity — use the OIDC ID token for login.
- Register exact redirect URIs (open redirects/loose matching let attackers steal codes).
- Prefer a hosted, standards-based identity provider over homegrown auth when you're a small team.

**Relevance to Sam**
- Supplier integrations (Viator, RateHawk) mostly use API keys or client-credentials-style server-to-server auth: keep keys/secrets in a secrets store on the backend only; never in front-end bundles or LLM prompts/logs.
- Tereka and Riselist logins: use a managed OIDC provider (Google sign-in, Auth0/Clerk/Supabase Auth etc.) with code + PKCE rather than rolling password storage; verify ID-token signature/iss/aud/exp.
- Riselist pulling creator data from TikTok, or any "connect your account" feature: that's delegated OAuth — minimal scopes, store refresh tokens encrypted, handle revocation.
- AI-generated OAuth code often uses the implicit flow, skips `state`/PKCE, or decodes JWTs without verifying signatures — check for these.

## Mike Pound (Computerphile) — Running an SQL Injection Attack (https://www.youtube.com/watch?v=ciNHn38EyRc)
**Core idea: if user input is concatenated into a SQL string, the user is writing your query; in a handful of requests an attacker goes from one stray quote to dumping every table, including password hashes — so data must never be able to become code.**

**Mental model**
- A search box in a PHP shop builds `SELECT ? FROM ? WHERE ? LIKE '%<input>%'`. The input should be data; with string concatenation, special characters (`'`, `;`, `--`) become SQL control structure. The attacker controls one slot but can rewrite the rest of the statement.
- Legal note: only test systems you own or are authorised to test (TalkTalk-style breaches end in prosecution and data-protection penalties).

**Attack walk-through (the methodology an automated tool follows)**
1. **Probe**: type a single `'`. A generic "server error" (instead of zero results or a product named O'Reilly) means the quote broke the query — injectable. Searching for `amme` returning "hammer" reveals the query uses `LIKE '%...%'`.
2. **Neutralise the tail**: input `';-- ` (quote, end statement, comment out the rest) returns every product — attacker now controls the query shape.
3. **Fingerprint the DBMS** (blind/timing technique): `hammer' AND 1=SLEEP(2)-- ` took 4 s for 2 matching rows, confirming MySQL (`SLEEP`; SQL Server uses `WAITFOR DELAY`; Postgres has its own). Timing works even when nothing is displayed — "blind SQL injection"; it can also probe whether a named table exists.
4. **Column count + UNION**: results show 3 columns; `... UNION SELECT 1,2,3 FROM dual` appends a row "1 2 3" to the hammer list. Visible output = catastrophic, because any query result can now be printed.
5. **Enumerate schema**: `UNION SELECT table_name, table_schema, 3 FROM information_schema.tables` lists every table in every database on the server (blog tables, `stock`, `users`); then `information_schema.columns WHERE table_name='users'` reveals `user_id, user_login, user_hash, user_type`.
6. **Exfiltrate**: `UNION SELECT u_login, u_hash, u_type FROM users` dumps logins, password hashes and which account is admin — the obvious target for offline cracking.
- About three requests in total once automated; the TalkTalk breach used a Python script doing exactly this in bulk.

**Mechanisms worth remembering**
- UNION stacks rows from two queries with equal column counts (vs JOIN, which merges columns). Pad with constants (`3`) to match counts.
- `information_schema` is the metadata catalogue every attacker reads first; a shared DB server means one injectable app exposes all co-hosted databases.
- Verbose errors and timing differences are oracles.
- Hashes leak too: many sites hash weakly, so a hash dump is often a password dump. (note: use a slow, salted password hash — Argon2id/bcrypt/scrypt.)

**Defences**
- Parameterised queries / prepared statements are the primary fix; escaping/sanitising input is secondary.
- **Second-order injection**: a malicious value (e.g. a username) is escaped correctly on insert and stored safely, but later read from the DB and concatenated into another query (e.g. during password change), where it executes. Data from your own database is not automatically safe — parameterise every query, not just ones touching direct user input.
- Implied extras: generic error pages (no SQL errors to users); least-privilege DB accounts that can't read `information_schema` beyond their schema or other apps' databases; separate databases per app; cap result counts.
- Session tokens: the closing remark — a stolen cookie lets the attacker impersonate you to the shop (basket, address, card) — links injection/XSS to session theft.

**Relevance to Sam**
- Data pipelines that build SQL with f-strings (DuckDB, Postgres, SQLite) over GeoNames/OSM/supplier data are injectable by content: a place name or hotel description containing `'` either breaks the job or runs code. Supplier strings and scraped web text count as attacker-controlled; always bind parameters (`?`/`%s`), including in DuckDB and pandas `read_sql`.
- Second-order risk is real in ETL: values ingested once, reused later in dynamic SQL (e.g. table or column names built from data) — allow-list identifiers, since they can't be parameterised.
- LLM-generated SQL (text-to-SQL or AI-written code) is the same pattern: run with a read-only, least-privilege role, statement allow-lists and row limits.
- Tereka/Riselist: use an ORM or parameterised queries; store passwords with Argon2/bcrypt via a vetted library or a managed auth provider.

## McKenzie (Aikido Security) — What is SSRF? Server-Side Request Forgery Explained (With Real Attack Example) (https://www.youtube.com/watch?v=-KSOFjNOOMw)
**Core idea: SSRF is a confused-deputy attack — the attacker can't reach your internal network, but your server can, so a user-controlled URL turns your server into the attacker's employee who walks past the firewall.**

**Mental model**
- Bank analogy: you're stopped by the guard (firewall), so you ask a friend who works there (vulnerable app) to fetch the files. Network = bank, guard = firewall, employee = server that fetches URLs.
- Same root as SQLi, XSS and command injection: the server trusts data the user can manipulate. Here the data is a URL.
- You rarely *intend* to accept URLs; they sneak in via features: "generate a PDF from this page", "preview this link", "fetch avatar from URL", image proxies. The browser UI is just one client — attackers send the raw request directly and change the parameter.

**Mechanism**
- Server code like `requests.get(user_url)` with no checks. Attacker swaps an expected image URL for `http://localhost/admin`, an internal IP, or the cloud metadata endpoint (on AWS `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>`).
- Prime target: credentials. Cloud instances expose temporary IAM credentials via the metadata service so the machine can call cloud APIs; a fetcher that reaches it hands those keys to the attacker.
- **Capital One (2019)**: attacker found a misconfigured web application firewall running on AWS, used SSRF to make it query the internal metadata endpoint, obtained temporary AWS credentials for its role, then used them to read S3 buckets — ~100 million credit applications, US SSNs and Canadian SINs, tens of thousands of bank account numbers; large fines and customer settlements. (The presenter jokes about the cost; the real figure was an $80M regulatory penalty plus a ~$190M class settlement — note: from general knowledge, not the video.)
- Root design smell: one over-privileged "do-everything" backend service that fetches whatever it's told, rather than narrow services each doing one thing.

**Defences (in priority order)**
1. Don't let users supply URLs at all where avoidable: if you fetch avatars from one service, hard-code that service and accept only an ID/path.
2. Allow-list destinations (specific hosts/IPs), schemes (only `http`/`https` — block `file://`, `gopher://`, etc.), and ports.
3. Harden the credential source: on AWS require IMDSv2 (session-token, PUT-first) so a simple GET-based SSRF can't read metadata credentials; give the instance role least privilege so stolen keys reach little.
4. Network controls: firewall/egress rules so the fetcher can't reach internal ranges or the metadata IP; least privilege everywhere.
5. Split services: a fetcher with no secrets and no internal network access is worthless to hijack.
- (note — standard gotchas the short video doesn't cover: validate the *resolved* IP, not the hostname, and pin it for the actual connection to beat DNS rebinding; block private/loopback/link-local ranges incl. IPv6 and decimal/octal IP encodings; disable or re-validate redirects at every hop; set timeouts and size caps.)

**Relevance to Sam**
- The Drive Button / LLM pipeline that fetches external blog pages is an SSRF surface by design. Run the fetcher as an isolated, credential-free worker (separate container/VPC/egress proxy), resolve-and-check IPs against private ranges, block 169.254.169.254, require IMDSv2 if on AWS, cap redirects, size and time, and only allow http/https.
- Anything that fetches supplier image URLs, creator profile links (Riselist), or "import from link" features is the same pattern.
- Fetched content is also untrusted *text*: the same pages can carry prompt injection aimed at the LLM — treat the model's output as untrusted too and never give the fetch-capable agent secrets or write tools it doesn't need.
- AI-written fetch helpers almost always call `requests.get(url)` blindly — audit every such call.

