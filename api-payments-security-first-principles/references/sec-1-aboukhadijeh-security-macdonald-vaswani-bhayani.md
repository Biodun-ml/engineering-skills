# sec-1 — cross-cutting principles

1. **Assume one layer has already failed.** Design auth so it holds even when the attacker has the password, the DB dump, or an XSS foothold. Defence in depth means MFA, slow hashing, rate limits and step-up re-auth all at once. (Feross L13, Feross L01)
2. **Never trust the client.** Attackers send requests your UI never would (curl, scripts). Hiding a button is not access control. Every rule and every price is checked on the server. (Feross L01; Aikido A01)
3. **Authenticated ≠ authorised.** Check ownership on *every* object access, because IDOR/BOLA is still the #1 vulnerability and the #1 bug-bounty finding. Look permissions up server-side from an opaque session rather than baking them into tokens you can't revoke. (Feross L13; Aikido A01; CNCF JWT)
4. **Don't build your own auth if you can avoid it.** Use a trusted provider. If you must build it: argon2/bcrypt with salt (and pre-hash to get around bcrypt's 72-byte cap), length over complexity, a breached-password check, no forced rotation, and MFA for financial apps. (Aikido A07; Feross L13)
5. **Leak nothing in auth responses.** Use the same message, status code *and timing* for unknown user, wrong password and locked account. Apply this to login, reset and signup, or rate-limit hard so enumeration can't scale. (Feross L13)
6. **Rate-limit and detect automation on every auth and abuse path**, keyed per account and per IP. Make the limit about 10x above the most extreme real user. CAPTCHAs are weak because they're solved for about $0.50 per 1000. Quotas must be keyed on something attackers can't mint for free. (Feross L13, Feross L01 localStorage bug; Aikido A07)
7. **Re-authenticate (sudo mode) before sensitive changes** such as email, password, payout or shipping address, or API keys. This limits the damage from XSS, CSRF and session theft. (Feross L13)
8. **JWT = readable, signed claim.**
   - Pin the algorithm on the verifier and reject `none` or header-chosen algorithms. Verify the signature, then `exp`/`iss`/`aud`.
   - Use long random secrets, or asymmetric keys.
   - Keep keys in a KMS or Vault.
   - Use short lifetimes because revocation is hard.
   - For a monolith, opaque sessions are simpler and safer. (CNCF JWT)
9. **Bearer tokens and cookies are the new passwords**, because stealing a token gets around MFA. Use HttpOnly + Secure + SameSite cookies, never localStorage, and never log tokens. (CNCF JWT; Feross L13)
10. **Keep secrets out of logs.** Request-logging middleware dumped plaintext passwords for years at Facebook and Twitter. Redact passwords, tokens, API keys and card data before anything is logged. A stray card number in a log or error tracker also pulls that system into PCI scope. (Feross L13; ControlCase)
11. **Separate command from data.** Injection, XSS and URL bugs all come from mixing them. Use parameterised queries, encode output for its context, and enforce on the backend. (Feross L01; Aikido A05)
12. **Logic flaws need threat modelling, not scanners, and AI coders make them by leaving invariants out.** Write the invariants down explicitly (no negative amounts, refund ≤ charge, server-side price) and test them. (Aikido A06)
13. **The supply chain includes your IDE and CI.** Hijacked npm packages (debug/chalk, about 2.6B weekly downloads), typosquats (one study's packages ran on about 17k domains), and malicious VS Code/Cursor extensions all steal local secrets. Use SCA plus malware scanning, few dependencies, and publish only from CI. (Aikido A03/A08; Feross L01)
14. **Default-private infrastructure.** Public S3 buckets and leftover sample apps cause breaches. Scan IaC before deploy and the live cloud (CSPM) after. (Aikido A02; Feross L13)
15. **Log *and* alert, and fuzz for weird input.** Unwatched logs don't help. Unicode, long payloads and malformed input must fail safely. (Aikido A09/A10)
16. **PCI scope = anything that stores, processes or transmits card data, plus whatever is connected to it.** The cheapest compliance is never touching the PAN: use PSP hosted fields or redirects and store only tokens (SAQ A rather than SAQ D). Never store CVV. Segment networks, and require AOCs from vendors. (ControlCase; note: current version is v4.0.1)
17. **A timeout means "unknown", not "failed".** Every money-moving or booking call takes an idempotency key generated once per user intent. Enforce it with a unique DB constraint, store the response, check status by ID before retrying, and dedupe webhooks. (Arpit Bhayani)
18. **Don't auto-retry what isn't idempotent.** For low-stakes actions, surfacing the error can be simpler than building idempotency. For payments you need both: idempotency *and* no blind retries. (Arpit Bhayani)

---

## Feross Aboukhadijeh (Stanford CS253) — Web Security Lecture 13: Authentication (https://www.youtube.com/watch?v=lgmGSYwJc20)
**Design authentication assuming the attacker already has the user's password: hash slowly with salt, limit guessing, leak nothing, and add a second factor.**

**Mental models / first principles**
- Authentication = proving who the user is. Authorization = deciding what that known user may do. Keep them separate (NIST says so explicitly). Hand out an opaque session ID and look up permissions on each request; do not mint a token that encodes "allowed to do X, Y, Z", because you cannot expire or update it when permissions change. (This foreshadows the JWT pitfalls in the CNCF talk.)
- Factors: something you know (password), something you have (phone, hardware key), something you are (biometric). More factor categories means more confidence. ATM = card (have) + PIN (know).
- You cannot reset biometrics if they leak, so good systems keep them on the device. The biometric only unlocks a local keystore, and a crypto key proves identity to the server. WebAuthn works this way.
- Capability URLs (the unguessable token in a Google Docs share link) are a form of authorization: holding the URL is the permission.
- Guiding question for the lecture: how do you stay secure when the password is already known? Defence in depth.

**Password policy (current NIST-style guidance)**
- Old rules are counter-productive: forced rotation every 90 days (users just bump a trailing digit), mandatory symbol/case mixes, password history, minimum password age (stops users fixing a leaked password right away), blocking paste (breaks password managers), weak hint questions (one airline used 8 drop-down answers), on-screen keyboards, "security images" (an attacker can fetch the victim's image by entering their username, and a study found 72-73% of users typed the password with the image removed).
- Do this instead: minimum length 8 (bare floor), allow up to about 64 or more characters, allow any Unicode and whitespace, check new passwords against breach corpora (e.g. Have I Been Pwned), drop forced rotation, rate-limit login, and require MFA for sensitive apps (financial) while strongly encouraging it elsewhere.
- Length beats complexity. Four random dictionary words beat "Tr0ub4dor&3" even if the attacker knows the scheme and the word list. Under about 12 characters is crackable by a motivated attacker.
- Set an upper bound. Unlimited length enables a password-hashing DoS (slow hash on a huge input). Pre-hashing with a fast hash also fixes this.
- **bcrypt gotcha:** input is capped at 72 bytes and the rest is silently truncated. With multi-byte Unicode the usable length is even shorter. Fix: pre-hash (e.g. SHA-256, then encode) before bcrypt. Never silently truncate in your own code.
- Never log request bodies that carry passwords. Facebook and Twitter both logged plaintext passwords for years. Logging middleware has to redact them.
- Legacy horror stories: banks capping passwords at 10 characters, and one bank mapping letters to phone-keypad digits, which collapsed the keyspace.

**Online guessing attacks and defences**
- Brute force (one account, many passwords); credential stuffing (username/password pairs from another site's breach); password spraying (one common password across many accounts).
- Rate limit per username and per IP (e.g. express-rate-limit). Respond by delaying or returning 429. Rule of thumb: set the limit about 10x above the most extreme real user, since many people can share one corporate NAT IP, then tune.
- CAPTCHAs are weak. Text CAPTCHAs are broken by ML (a 2014 Stanford paper beat all schemes it tested; anything over 1% solve rate counts as broken). Human solving farms cost about $0.50 per 1000. Attackers can also relay a CAPTCHA to visitors of their own site. Behavioural, score-based systems (reCAPTCHA v3) work better but punish Tor and privacy users.
- Heuristics: one IP trying 10 accounts in a minute, new device (no known-device cookie), new country, or an IP on a public bad-IP list. Trigger step-up MFA on these instead of asking every time.
- **Re-authenticate before sensitive actions** (change email or password, add shipping address, add SSH key or collaborator; GitHub's "sudo mode"). This limits the damage from XSS, CSRF, session fixation and a laptop left unlocked.

**Account enumeration ("response discrepancy information exposure")**
- Use one generic message for wrong user, wrong password, locked and disabled accounts. Apply it to login, password reset ("if the email exists, we sent a link") and signup.
- Match HTTP status codes across cases too, not just message text.
- Timing side channel: returning early when the user does not exist skips the slow hash and the DB call, so response time reveals which accounts exist. Run the same work on both paths (hash anyway). Measure it empirically.
- Trade-off: generic errors frustrate real users. The alternative is friendly messages plus tight rate limits so enumeration cannot scale. Choose by sensitivity. For a bank or a money app, a list of confirmed customers makes phishing much easier.

**Password storage**
- Never store plaintext; breaches come from misconfigured servers, SQLi or command injection followed by pivoting, and public S3 buckets.
- A plain SHA-256 is not enough: identical passwords give identical hashes, and attackers can precompute lookup tables.
- Salt: a random value of 16-32 bytes, unique per user, stored next to the hash (it is not secret), and concatenated before hashing.
- Use a deliberately slow, salted KDF: bcrypt with a cost/rounds parameter, whose output `$2b$10$<salt><hash>` embeds the algorithm, cost and salt. Compare using the library's compare function. (note: today argon2id or scrypt are also standard picks. Keep the 72-byte cap in mind if you use bcrypt.)
- Reality check (Microsoft, 2019): a $20k rig does about 100 billion SHA-256 guesses per second, and a list of 500M real leaked passwords cracks about 70% of users. Even good hashing does not save weak or reused passwords, which is why you need MFA. Microsoft claims MFA blocks 99.9% of account compromises.
- Proactive move: check your users against breach data and force a reset on those whose credentials have leaked.

**TOTP mechanism (RFC 6238 style)**
- The server generates random per-user secret bytes, stores them, and shares them via a QR code (or manual entry). The app needs no network.
- counter = floor(unix_time / 30); HMAC(secret, counter), then dynamic truncation, then mod 10^6, giving a 6-digit code. The server computes the same value and compares (allow a small clock-skew window).
- The secret is a shared secret, so it is sensitive in the DB as well. Encrypt it at rest.

**Rules of thumb / checklist**
- TLS everywhere. Generic auth errors with consistent status codes and timing. Rate limiting on every auth route, including reset and signup. bcrypt or argon2 with pre-hash. Breached-password check. Redact secrets from logs. Step-up re-auth for sensitive changes. MFA, mandatory for financial apps.

**Relevance to Sam**
- Tereka (money planner) counts as "financial": require or strongly push MFA and re-auth for changes to email, password or payout details, and use generic errors so nobody can list which Nigerians use it.
- AI-written auth code tends to produce early returns (timing leaks), different error strings, and request-logging middleware that dumps bodies. Review for all three.
- Request logging in API integrations: redact passwords, and by extension supplier API keys and card data, before anything goes to logs.
- OWASP API2 (broken authentication) and API4 (resource consumption): rate-limit auth endpoints.

## Feross Aboukhadijeh (Stanford CS253) — Web Security Lecture 01: What is Web Security? HTML & JavaScript Review (https://www.youtube.com/watch?v=5JJrJGZ_LjM)
**Think like an attacker while you build. The web runs untrusted code inside one shared user context, so every layer needs defences that still hold when another layer fails.**

**Mental models / first principles**
- **Attacker mindset + defender mindset.** For every design and every line of code, ask how someone could break it. A system can work perfectly and still be insecure: correctness and security are separate properties.
- **Attack is easier than defence.** The attacker needs one hole. The defender can never prove the code is secure, and a bug may be in a dependency. So design for defence in depth: if one control fails, another still protects you.
- **Vulnerability vs exploit.** A vulnerability is a flaw. An exploit is weaponised code that uses it. There are separate markets for finding bugs, weaponising them, buying compromised machines and cashing out (e.g. stolen cards → cloned cards → ATM withdrawals). Attackers are paid professionals, not hobbyists.
- What compromised machines are used for: DDoS for ransom or against competitors, crypto-mining, spam sent from reputable IPs, scanning for more victims, serving browser exploits to every visitor of a hijacked popular site, data theft (SSNs, card numbers, health data), ransomware (cryptocurrency made it easy to monetise), political hacks.

**Four views of "web security"**
1. **Browser/isolation view:** the same-origin policy, which the lecturer calls the most important concept in web security. Your bank tab must be shielded from a Wikipedia tab, and a third-party ad iframe must not touch the page around it.
2. **Server view:** you do not control the client. Anyone can write a script that sends a request your UI never would (curl). Hiding a button is not access control. Enforce every rule on the server. This is the root of OWASP API1/BOLA and mass assignment.
3. **Client-app view:** a crafted URL or stored content must not make your app run attacker code (XSS). Examples: a McDonald's site that leaked passwords when a link was visited, and stored XSS in WordPress that used a logged-in admin's session to create a new admin account, then reused those credentials on many WordPress sites. Getting a victim to load a URL is trivial (links, iframes, pop-ups), so "users won't click it" is no defence.
4. **User view:** social engineering (a DEF CON phone pretexting demo got SSNs read out over the phone) and tracking/fingerprinting. Ask: can the system stay safe even if the user is socially engineered? MFA is one answer.

**Supply chain**
- Typosquatting study: a student published packages named like popular ones that only phoned home. They ran about 45,000 times on about 17,000 domains, including .mil hosts. `npm install react` pulls in code from hundreds or thousands of authors, so nobody can audit it all. npm now blocks names one character away from popular packages. A cleverer attacker re-exports the real package so nothing looks broken.

**Why the web is hard to secure**
- The browser's job is close to impossible: run code from known-malicious strangers, open sockets, use the GPU, camera and microphone, show any media type, all safely.
- Strict backward compatibility ("don't break the web") means insecure old designs live on, especially around `<script>`, `<link>` and `<style>`, where most attacks happen.
- Powerful new APIs keep arriving and interact combinatorially. Example: the lecturer's localStorage bug. The 5 MB quota applied per subdomain, and subdomains are free, so iframes on 1.x.com, 2.x.com, ... could fill the user's disk. Quotas must be keyed on the right unit (registrable domain, not hostname). The same lesson applies to rate limits keyed on something attackers can mint cheaply (API4).
- HTTPS everywhere is now the norm. Two drivers: sniffing cookies on coffee-shop Wi-Fi (e.g. Firesheep) and the Snowden revelations of bulk interception.

**Background review (useful precision)**
- URL parts: scheme, host, port, path, query (`?k=v&k2=v2`), fragment (`#id`, which never reaches the server). Relative URL resolution depends on the trailing slash. A leading `/` resolves from the site root.
- **Core recurring problem: telling the command apart from user data.** Example: an `&` inside a query value has to be escaped. Injection, XSS and URL-building bugs all come from mixing the two.
- JavaScript language vs host APIs: the DOM comes from the browser, fs and net from Node. Most of the bad APIs live in the DOM. JS lets you override built-ins (historically even `undefined`). Old APIs such as window.open, moveTo and resizeTo still work.

**Rules of thumb**
- Never trust the client. Validate and authorise server-side.
- Expect attacker-controlled URLs and content to reach your users' browsers.
- Treat dependencies as attack surface: pin versions, check names carefully, keep the dependency count low.
- Practise responsible disclosure when you find bugs. Do not delete data or degrade service.

**Relevance to Sam**
- AI-generated code tends to pull in many packages and can invent package names. A hallucinated name an attacker has registered is a modern typosquat. Check every new dependency.
- For supplier APIs and booking endpoints, assume callers bypass the frontend: authorise each booking ID and price on the server.
- The Drive Button pipeline injects content into third-party blog pages. That is the ad-iframe/mashup situation, so isolate the widget (iframe or sandbox) and never let host-page data execute as code.

## Aikido Security — OWASP Top 10 2025: Your complete guide to securing your applications (https://www.youtube.com/watch?v=Jzr0Jdnq_EI)
**A quick tour of the 2025 OWASP Top 10, with the tool or practice that catches each category. Biggest takeaways: authorisation bugs are still #1, supply chain is now #3, and logic flaws need threat modelling because scanners (and AI coders) miss them.**

**What changed 2021 → 2025** (the list is refreshed about every 4 years)
- New: **A03 Software Supply Chain Failures** (replaces "Vulnerable and Outdated Components" and is wider in scope) and **A10 Mishandling of Exceptional Conditions**.
- Security Misconfiguration rose from #5 to #2. Injection fell from #3 to #5. Insecure Design is now #6. Cryptographic Failures dropped because tools find them easily.

**The list, with mechanisms and fixes**
1. **Broken Access Control** (still #1). The user is authenticated, but the app never checks that they may touch *this* object.
   - IDOR: change `id=123` to `124` in a URL or, more often, in an API call. The presenter says it is the #1 bug class in bug bounties. (note: IDOR stands for *Insecure* Direct Object Reference; the video says "indirect".)
   - Related forms: privilege escalation to admin; missing object-level authorisation (API1 BOLA); missing function-level authorisation (an ordinary user calling an admin endpoint); path traversal (`../` to reach config or env files holding secrets).
   - Found with SAST, DAST, or AI-driven pentesting.
2. **Security Misconfiguration**: the infrastructure layer.
   - Public S3 buckets: storage left public because nobody made it private explicitly; the cause of countless breaches.
   - Sample or demo apps left deployed with known vulnerabilities, which become a stepping stone into the cloud account.
   - Tools: **IaC scanning** catches problems before deploy; **CSPM** (cloud security posture management) inspects the live cloud. You need both.
3. **Software Supply Chain Failures.** Open-source dependencies are about 75-90% of an app's code.
   - 2025: debug, chalk and related packages were hijacked through a single maintainer phishing campaign, about 2.6 billion weekly downloads affected.
   - The supply chain also covers the IDE: malicious VS Code extensions, especially in the marketplaces of forks like Cursor and Windsurf, steal local secrets. CI/CD pipelines are part of it too.
   - Tools: SCA (known CVEs in dependencies) plus **malware scanning** of packages and extensions, since a malicious package has no CVE.
4. **Cryptographic Failures.** Weak hashes (MD5), weak keys or seeds, sensitive data unencrypted, especially in transit. Easiest to catch in code with SAST.
   - (note: MD5 is broken, not "difficult but possible". For passwords, any fast hash, including SHA-256, is wrong; use bcrypt, scrypt or argon2.)
5. **Injection.** Untrusted data gets read as code: SQL, NoSQL, XSS, command or code injection.
   - Fixes: parameterised or prepared statements, plus escaping or encoding at the point of use, on the backend. A frontend-only check is bypassed by editing the request directly.
   - (note: the video calls this "escaping at the request level". The stronger principle is output encoding by context plus parameterisation, not input filtering.)
   - Tools: SAST, and DAST, which runs scripted attacks.
6. **Insecure Design**: logic flaws in a correctly coded system.
   - Example: Alice sends Bob −$20. Does that pull money from Bob? A real Coinbase bug allowed effectively unlimited money.
   - Error messages that leak internals also count.
   - Scanners rarely find these. **AI/vibe coding is especially bad here**: it fills in unstated assumptions, and if you never say "no negative amounts" it will not enforce it.
   - Remedy: **threat modelling** (e.g. STRIDE). Put on a black hat and attack your own flows.
7. **Authentication Failures** (logging in, as opposed to #1, which is what you may do once logged in).
   - Brute force, dictionary attacks and credential stuffing. Defences: lockout or attempt counters, escalating CAPTCHAs, MFA, anything that stops automation.
   - (note: the presenter mixes up credential stuffing, which replays leaked *username+password pairs* from other breaches, with dictionary attacks using a password list such as RockYou. RockYou has about 14M passwords, not 3M.)
   - Default credentials (WordPress admin/admin, per the presenter). Weak password-reset flows built on guessable security questions such as mother's maiden name or best friend, which are easy to find on social media.
   - **Strongest advice: don't build your own auth.** Use a trusted provider (Okta, Auth0, or open-source equivalents) for SSO and MFA. If you must roll your own, test it hard with SAST, DAST and Burp Suite.
8. **Software or Data Integrity Failures**: trusting data or artefacts without checking where they came from.
   - Unsigned or unverified cookies that the client can swap to raise privileges.
   - Packages published with a stolen developer token instead of only through the CI pipeline. Fix: allow publishing only from a trusted path (e.g. GitHub Actions trusted publishing or provenance).
   - Signing and hash verification. Largely a pipeline-design problem.
9. **Logging & Alerting Failures** (renamed to stress alerting). Log auth attempts and injection-like payloads, *and* alert on them. Logs nobody watches don't help. Missing logs leave you blind and make debugging painful.
10. **Mishandling of Exceptional Conditions**: odd inputs such as non-Latin scripts, Unicode, very long payloads, malformed requests.
    - Does the app fail safely, log correctly and avoid crashing or leaking?
    - Tool: **fuzzing**, i.e. chaotic random input, unlike DAST's structured attacks.

**Rules of thumb**
- Authenticated is not the same as authorised: check ownership on every object access.
- Default-private storage. Remove sample apps. Scan IaC before deploy and the live cloud after.
- Use SCA plus malware scanning on dependencies *and* editor extensions.
- Make threat modelling an explicit step for money-moving logic.
- Buy auth, don't build it.

**Relevance to Sam**
- Tereka and booking flows: write down the business invariants and test them, because AI-written code will not invent them. Examples: no negative amounts or quantities, the price recalculated server-side, a refund never larger than the charge, currency fixed per booking.
- Supplier API keys (Viator, RateHawk) sit in local env files. Malicious IDE extensions in Cursor or Windsurf-style editors and compromised npm or PyPI packages target exactly those. Use few extensions, scoped keys, and a secrets manager.
- Fuzz the GeoNames and location pipelines and the LLM page-fetcher with weird Unicode and huge payloads (A10 and API4).
- Build alerting, not just logging, for auth failures and supplier-call anomalies.

## Bruce MacDonald (InfraHQ, CNCF) — Handling JWTs: Understanding Common Pitfalls (https://www.youtube.com/watch?v=7ybmox6KQ8c)
**A JWT is a signed, readable claim. All of its security comes from the server verifying the signature with an algorithm and key the server chose itself, and from storing both the key and the token safely. Often a plain session token is the better tool.**

**Mental model**
- Bar analogy: a bartender checks that the ID comes from a trusted issuer (the government), checks it is not forged or expired, and only then trusts what it says. A JWT is the same thing for the web: an identity document issued by a trusted authority (Google, Microsoft, or your own auth service) that other services accept without calling the issuer.

**Format**
- Three base64url parts, `header.payload.signature`.
  - Header: `{"alg":"HS256","typ":"JWT"}`.
  - Payload: claims, e.g. `email`, `iat` (issued-at, Unix time) and `exp` (expiry, Unix time). Reject tokens with `iat` in the future or `exp` in the past.
  - Signature: an integrity check over header and payload.
- **Base64 is encoding, not encryption.** Anyone holding a token can read its payload. Never put secrets or PII you wouldn't show the user into a JWS. If you need confidentiality, use JWE (encrypted JWT).

**Signing options**
- **Symmetric (HS256, HMAC + shared secret).** Simple and common, but every verifier holds the same key and can therefore *mint* tokens. With many parties you can't tell who issued a given token, and the more copies of the key exist, the more places it can leak.
- **Asymmetric (RS256/ES256).** Only the auth service holds the private key. The public key is published (e.g. at a JWKS endpoint), so any microservice can verify but none can forge. The speaker prefers this for multi-service setups.
  - (note: the talk describes RSA signing as "encrypting the hash with the private key". That is a simplification. Treat signing and verifying as their own primitives, and use a library.)
- Flow: the client logs in to the auth service, which checks the password and returns a signed JWT. The client then calls service X with `Authorization: Bearer <jwt>`. X fetches the public key, verifies the signature, checks `exp`/`iat`, and then acts.

**Attacks and pitfalls**
1. **`alg: none` / algorithm confusion.**
   - The naive way to verify reads `alg` from the token header and lets the token decide how it gets checked. The RFC defines `none` (unsigned).
   - Auth0 had this bug in 2019: a differently-cased "nOnE" got past their filter.
   - Demo: a hand-built header `{"alg":"none"}` plus payload `email: admin@example.com` was accepted.
   - **Fix: the verifier pins the allowed algorithm(s), e.g. only RS256, and never trusts the header's choice.** Check that your library rejects `none` by default.
   - (note, not in the talk but closely related: the RS256→HS256 confusion attack, where the attacker HMAC-signs with the *public* key. Pinning the algorithm *and* the key type defeats it.)
2. **Weak HMAC secrets.**
   - A short secret (4 chars) was brute-forced offline with jwtcrack in about 10 seconds on a MacBook. The attacker then re-signs any claims they like (jwt.io makes this trivial).
   - Fix: a long, random secret sized to the algorithm, i.e. at least 256 bits for HS256. Never use a human password.
3. **Token theft is rising because MFA works.**
   - Stealing a bearer token skips both the password *and* the MFA step until the token expires.
   - Storage options:
     - Cookie with **HttpOnly** (no JS access, so XSS can't read it), **Secure** (HTTPS only), and **SameSite** (Strict or Lax unless SSO redirects need otherwise).
     - Or keep it in memory only. Safer, but users must log in again after a page refresh.
   - (note: localStorage is readable by any XSS, which is why the speaker avoids it.)
4. **Key management is the real crown jewel.**
   - Keep signing keys in an HSM or KMS (e.g. AWS KMS/CloudHSM) or HashiCorp Vault, not copied into env vars on every host.
   - Benefits: access is audited (who used the key) and rotation happens in one place after a compromise.
5. **Revocation.**
   - A JWT is stateless, so it stays valid until `exp`. Logout, a ban, or a permission change can't take effect early unless you add a denylist, and that throws away the stateless benefit.
   - Keep JWT lifetimes short (minutes), and pair them with refresh tokens if needed.

**Decision: JWT vs session token**
- JWTs make sense when many distributed services must verify identity without a network call to a central auth service (lower latency, no single bottleneck).
- For a monolith or a simple app with fast DB access, use **opaque random session IDs** looked up server-side. They are simple and can be revoked instantly. (This matches Feross's advice: don't put authorisation inside tokens you can't revoke.)

**Checklist**
- Pin the algorithm.
- Verify the signature *before* reading any claims.
- Check `exp`, `iat`/`nbf`, and also `iss` and `aud` (so a token from another issuer or meant for another app is rejected).
- Use long random HMAC secrets, or prefer asymmetric signing.
- Keep keys in KMS or Vault and rotate them.
- Store tokens in HttpOnly + Secure + SameSite cookies.
- Use short expiry.
- No secrets in the payload.
- Use a well-maintained library, never hand-rolled parsing.

**Relevance to Sam**
- Tereka and Riselist are effectively monoliths, so server-side sessions (or a managed auth provider) are simpler and safer than JWTs.
- If AI-generated code produces `jwt.decode(token)` without verification, or `algorithms=` taken from the header, that is a critical bug. In PyJWT, always pass `algorithms=["RS256"]` (or `["HS256"]`) plus `audience` and `issuer`.
- Supplier integrations that issue OAuth access tokens: cache them server-side, never expose them to the browser, and respect their `exp`.
- An LLM pipeline that fetches pages must never forward your tokens or cookies to arbitrary URLs.

## Kishor Vaswani (ControlCase) — Webinar: PCI DSS Compliance Checklist (https://www.youtube.com/watch?v=UWlkiNB6cMg)
**PCI DSS applies to every system that stores, processes or transmits cardholder data. Its 12 requirements are ordinary security hygiene. The cheapest way to comply is to keep card data out of your systems in the first place.**
(A light vendor webinar: a high-level summary of PCI DSS v3.2.1-era structure. note: PCI DSS v4.0 replaced v3.2.1 in March 2024, and v4.0.1 is current. v4 keeps the 12-requirement structure but renames some items, e.g. Req 1 is now "network security controls" rather than firewalls, and Req 5 is "anti-malware". It also adds e-commerce script controls (6.4.3) and payment-page tamper detection (11.6.1).)

**What and who**
- Created in 2006 by the card brands (Visa, Mastercard, etc.) and maintained by the PCI Security Standards Council. It is one of a family of standards, alongside PA-DSS, P2PE, PIN and 3DS.
- Protected data:
  - **Cardholder data**: PAN, cardholder name, expiry.
  - **Sensitive authentication data**: full track/magstripe data, CVV/CVC, PIN and PIN blocks.
  - (note: SAD must never be stored after authorisation, even encrypted. Only the PAN etc. may be stored, and only if there is a business need.)
- **Scope rule:** any entity or system that stores, processes or transmits card data is in scope. That covers merchants, service providers, acquiring banks and the card brands, all assessed by QSAs (Qualified Security Assessors).

**6 principles → 12 requirements**
1. Build and maintain a secure network
   - R1: firewalls and a DMZ, segmenting internet, DMZ (web servers) and internal network; regular ruleset reviews.
   - R2: secure configuration baselines and hardened standard images; no vendor defaults.
2. Protect cardholder data
   - R3: protect it at rest. Storing it is allowed only with a business need.
   - R4: encrypt it in transit over public networks with strong cryptography.
3. Vulnerability management
   - R5: anti-malware, installed and kept updated. The speaker calls it the easiest requirement and says it is clearly insufficient on its own.
   - R6: secure development and testing for application vulnerabilities (the OWASP categories).
4. Access control
   - R7: need-to-know access.
   - R8: identification and authentication: passwords, prompt deprovisioning of leavers, and remote-access controls (MFA).
   - R9: physical security: CCTV, badges, visitor logs. In the cloud this is **inherited from the cloud provider**.
5. Monitor and test
   - R10: logging and monitoring: a SIEM, synchronised clocks, intrusion detection, incident response.
   - R11: testing:
     - internal and external vulnerability scans quarterly (external scans by an ASV);
     - wireless access point reviews quarterly;
     - penetration tests (internal, external, application and segmentation) at least annually, sometimes every six months.
6. Policy
   - R12: a documented security policy that staff are trained on and acknowledge. Third parties and vendors must comply. Compliance is monitored and there is an incident-response process.

**Consequences and cost**
- Fines (the speaker's rough figure is $5k–100k, but it varies widely), lost revenue, reputational damage, and lost contracts after a breach.
- Cost levers:
  - automation of evidence collection, data discovery and scanning;
  - "assess once, comply to many" (map PCI controls to ISO 27001 and the NIST CSF);
  - **continuous compliance**: quarterly checks on about 20–25 high-risk items (scans, firewall reviews, data discovery, re-scoping, log health) rather than a once-a-year scramble.
- During COVID, the PCI Council allowed remote assessments.

**Scope-reduction takeaways** (the principle comes from the talk's scope rule; the tactics are standard practice)
- Every system that touches the PAN pulls the whole connected network into scope. So the main design goal is that **card data never touches your servers**:
  - Use the PSP's hosted payment page, redirect, or iframe/hosted fields (Stripe Checkout or Elements, Paystack or Flutterwave inline/redirect). The merchant then usually qualifies for the shortest self-assessment (SAQ A) instead of SAQ D.
  - Store only PSP tokens and customer IDs, never the PAN or CVV.
- Use network segmentation to separate any card-handling component from everything else, and pentest that segmentation.
- **Logs:** check that request logging, error trackers and LLM prompts never capture card fields. Any of these silently pulls those systems into scope.
- **Third parties:** a supplier or PSP that handles cards for you must itself be PCI compliant (R12 covers vendor management). Ask for their AOC (Attestation of Compliance).

**Relevance to Sam**
- **Trameter booking flows** with Viator or RateHawk: if Trameter is merchant of record, take card payment through a PSP's hosted fields and pass only tokens onward. Never proxy raw card numbers to a supplier through your own backend unless you accept full PCI scope.
- **Tereka**, a money *planner*: it should not need card data at all. Keep it that way, and don't add "link your card" features without a PSP or aggregator in between.
- AI-written payment code often posts the card form to your own API "for convenience". That single choice puts the whole stack in scope. Reject it.

## Arpit Bhayani — Build a robust Payments service using Idempotency Keys (https://www.youtube.com/watch?v=m6DtqSb1BDM)
**Any operation that might be retried must be idempotent. Generate one ID for the transaction before the first attempt, send that same ID on every retry, and have the server check its status before acting so the second call becomes a no-op.**

**Mental models / first principles**
- **Idempotent:** running an operation N times leaves the same state as running it once. The video's example: double-tapping an Instagram like still adds one like.
- Examples where it matters: posting the same tweet twice, placing the same order twice, sending a chat message twice, and above all **charging money twice**.
- **Retries are the root cause.** Duplicates appear because something retried:
  - a user double-click;
  - client-side auto-retry (JS, Android, iOS);
  - service-side retry after a timeout or crash recovery;
  - background sync workers.
- **The classic failure:**
  1. The payment service asks the gateway to send $20k from A to B.
  2. The gateway completes the transfer.
  3. The response is lost (network partition) or the payment service crashes before recording it.
  4. The service sees no confirmation and retries, and the gateway transfers again. A loses $40k.
- **The key point:** a timeout does *not* mean the operation failed. Its outcome is unknown.

**Decision: do you need idempotency at all?**
- Arpit's pragmatic advice: idempotency adds checks on both backend and frontend, so not every endpoint needs it.
- If the product allows, **don't auto-retry**. Show the error and make the user start the flow again (an explicit retry, not an implicit one). Many real systems work this way, e.g. a failed Instagram like just shows an error.
- (note: for money movement, an explicit user retry still risks a duplicate when the first attempt actually succeeded behind a timeout. The user retries, and the server cannot tell it is the same intent. So payment endpoints should be idempotent *and* avoid blind auto-retry. Treat "skip idempotency" as advice for low-stakes actions only.)

**Mechanism: "check and update" with a common identifier**
1. **Create the ID first.** Before any money moves, the payment service gets a `payment_id` (e.g. `p1729`) from the gateway, or generates one itself.
2. **Persist it everywhere:**
   - the client (JS, Android, iOS) holds it;
   - the payment service stores it in its DB;
   - the gateway records it against the actual transfer.
   - This one ID ties all three stages together.
3. **Send it on every request:** "transfer $20k A→B *for p1729*".
4. **On retry, send the *same* ID** (never mint a new one). The gateway looks up p1729, sees it is already processed, and returns "already completed" without moving money again.
- This is how real gateways are built: Stripe, PayPal and Razorpay hand you an ID at the start and you pass it on every later call. (note: Stripe also accepts an `Idempotency-Key` request header and replays the saved response for a repeated key.)
- Arpit's suggestion: build it locally, inject failures (drop responses, kill the service), and watch the duplicate disappear.

**Implementation details** (the video only hints at "extra checks that depend on your DB"; these are the standard ones)
- **Make check-then-act atomic.** Two concurrent retries can both see "not processed". Use one of:
  - a **unique constraint** on the idempotency key;
  - `INSERT ... ON CONFLICT DO NOTHING`, then act only if the row was inserted;
  - a row lock / `SELECT ... FOR UPDATE`.
- **Track states:** `pending → processing → succeeded/failed`. A retry that finds `processing` should wait or poll. It must not re-execute.
- **Store the response** together with the key, so a replayed request gets the identical answer.
- **Scope keys per user/account** and bind each key to the request parameters. The same key arriving with a different amount should be rejected (HTTP 409/422), not silently replayed.
- **Expire keys** after a defined window (Stripe uses about 24h).
- **Reconcile:** if a gateway call times out, query its status by ID before retrying, and run periodic reconciliation jobs against gateway and supplier reports.
- Also make **webhook handlers** idempotent: dedupe on the event ID, because providers deliver at least once.

**Failure modes and gotchas**
- A new UUID per retry defeats the whole scheme. Generate the ID once, per user intent (e.g. when the checkout page loads), not per HTTP attempt.
- A check-then-act race with no DB constraint.
- Treating a timeout as a failure and retrying blindly.
- Retry loops spread across client, service and worker layers, which multiply duplicates.

**Rules of thumb**
- Every endpoint that moves money, books inventory or sends something irreversible accepts an idempotency key and enforces it with a unique index.
- Only retry calls that are idempotent. Use backoff with jitter.
- On an unknown outcome, check status by ID before retrying.

**Relevance to Sam** (payment safety, OWASP API6 sensitive business flows)
- **Trameter bookings with Viator/RateHawk:**
  - A lost response on "book" can produce two real reservations and two charges.
  - Send the supplier's partner or client reference field on every booking request, if the supplier offers one, and check booking status by that reference before retrying.
  - Persist your own booking intent ID *before* calling the PSP or the supplier.
  - If the PSP charge succeeds but the supplier booking fails, you need a compensating refund. That calls for a saga-style state machine, not ad-hoc retries.
- **Tereka/Riselist:** dedupe payment webhooks (Paystack, Flutterwave, Stripe) on the event or reference ID.
- **AI-written code** often wraps HTTP calls in generic retry decorators (tenacity, axios-retry). Audit every retry wrapper on a non-idempotent POST.
