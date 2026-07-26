# Security & Auth Interview Questions

Grouped by level, with model answers focused on trade-offs and failure modes
— the things interviewers actually probe. Work through after reading 01–03.
For the scenario questions at the end, practice answering out loud before
reading the model answer.

---

## Junior

### Q1: What's the difference between authentication and authorization?

**Answer:** Authentication establishes **who you are** (login, tokens,
sessions); authorization establishes **what you're allowed to do** (roles,
permissions, object ownership). They fail independently: an IDOR bug —
`GET /orders/991` returning someone else's order to a logged-in user — is an
authorization failure with authentication working perfectly. In code they're
usually separate layers: authn middleware resolves the identity, authz checks
run per action and per object. Strong answers point out that most real-world
access bugs are authz bugs, because authn is centralized once while authz
must be enforced on every endpoint and every object.

### Q2: How should passwords be stored, and why not SHA-256 with a salt?

**Answer:** With a slow, memory-hard, salted password hash — **argon2id**
(current recommendation) or **bcrypt** (cost 10–12), tuned so hashing takes
~100–300 ms. SHA-256 is designed to be *fast*, which is exactly wrong here:
an attacker with a leaked DB can try billions of salted-SHA-256 guesses per
second on GPUs, while argon2id's memory cost (tens of MiB per guess) makes
that hardware advantage collapse. Salts are per-user and automatic in modern
APIs — they prevent rainbow tables and identical-password detection, but do
nothing about speed, which is why salt alone isn't enough. Mention
`password_needs_rehash`-style upgrade-on-login and pairing with rate
limiting and breached-password checks for a complete answer.

### Q3: Walk me through the three parts of a JWT. What does the signature actually guarantee?

**Answer:** Header (algorithm, key ID), payload (claims: `iss`, `sub`,
`aud`, `exp`, custom), signature — three base64url segments. The signature
guarantees **integrity and authenticity**: the token was issued by the key
holder and hasn't been modified. It does **not** provide confidentiality —
anyone can decode the payload, so no secrets in claims — and it does not
guarantee the token is still *wanted*: a signed token remains
cryptographically valid after logout or compromise until `exp`. Verification
must also check the claims (`exp`, `iss`, `aud`) — a perfect signature on an
expired or wrong-audience token is still a rejection.

### Q4: What is CSRF and how do you prevent it?

**Answer:** Cross-site request forgery: an attacker's page makes the
victim's browser send a state-changing request to your site, and the browser
helpfully attaches the session cookie. Defenses, layered: `SameSite=Lax` or
`Strict` on the session cookie (stops most cross-site sends), anti-CSRF
tokens verified server-side with a constant-time compare, no state changes
on GET, and `Origin`/`Sec-Fetch-Site` checks as depth. Key nuance that
separates candidates: CSRF applies to **cookie-based auth** — an API
authenticated purely via `Authorization` header is inherently immune because
the attacker's page can't set that header cross-origin. And CORS is not a
CSRF defense: the forged request still executes; CORS only limits reading
the response.

### Q5: A teammate says "our API is protected because we configured CORS." What do you tell them?

**Answer:** CORS is a **browser** mechanism that relaxes the same-origin
policy so approved origins can read your responses — it protects users' data
from malicious *websites*, not your API from malicious *clients*. curl,
scripts, and any non-browser client ignore CORS entirely, and even in
browsers, simple cross-origin requests still reach and execute on the
server. So the API needs real authentication and authorization on every
request regardless. Also worth checking their config: reflecting the request
`Origin` while sending `Access-Control-Allow-Credentials: true` effectively
grants every website on the internet credentialed access — origins must be
an exact allowlist.

### Q6: What is SQL injection and what's the correct defense?

**Answer:** User input concatenated into SQL text becomes SQL: `email =
'x' OR '1'='1'` changes the query's logic; stacked or blind variants read or
modify anything the DB account can. The defense is **parameterized queries**
— input travels as data, never as query text — with escaping only as a
legacy fallback and allowlist-then-interpolate for identifiers (column
names, sort direction) that can't be bound. Strong answers add defense in
depth: the app's DB account should have least-privilege grants so even a
successful injection can't `DROP TABLE`, and ORMs are safe only until
someone uses the raw-SQL escape hatch with concatenation.

### Q7: Where should a web app store its auth token in the browser, and why?

**Answer:** Prefer an `HttpOnly; Secure; SameSite=Lax` **cookie** over
`localStorage`. `localStorage` is readable by any JavaScript on the page, so
a single XSS bug exfiltrates tokens for offline reuse from anywhere. With
`HttpOnly`, XSS can still *ride* the session while the page is open —
cookies don't make XSS harmless — but the credential itself can't be stolen
and replayed later, which meaningfully shrinks the blast radius. The cookie
choice reintroduces CSRF, so pair it with `SameSite` plus CSRF tokens. For
SPAs, a common pattern is the access token in memory and the refresh token
in an `HttpOnly` cookie scoped to the refresh endpoint.

---

## Senior

### Q8: JWT vs server-side sessions — defend a choice for a concrete system.

**Answer:** The model answer commits to a context. "For a single web
application with a monolith or a few services, I'd choose **server-side
sessions** in Redis: revocation is instant (logout, compromise, permission
change), the lookup is sub-millisecond so the scaling argument against
sessions is mostly folklore, and the code is simpler to get right. For a
platform where **many independent services** verify identity — public APIs,
mobile clients, an API gateway fronting dozens of microservices — I'd choose
**short-lived JWTs (5–15 min) with rotating refresh tokens**: services
verify locally against JWKS with no shared synchronous dependency, and
revocation happens at the stateful refresh endpoint so the compromise window
is bounded by access-token lifetime." What's being graded: you named the
real trade-off (revocation and staleness vs verification decoupling), gave
lifetimes, and didn't claim either is universally better.

### Q9: How does the JWT algorithm-confusion attack work, and how do you prevent it?

**Answer:** Two variants. `alg: none`: the header claims the token is
unsigned and a naive library skips verification. RS256→HS256 downgrade: the
attacker takes a token normally verified with an RSA **public** key,
rewrites the header to HS256, and signs it using the public key bytes as the
HMAC secret — a library that lets the *token* select the algorithm will
verify HMAC(public key) successfully, and since the public key is public,
the attacker can mint arbitrary tokens. Root cause: trusting attacker-
controlled input (the header) to choose the verification procedure. Fix: the
**verifier pins an explicit algorithm allowlist** (`WithValidMethods` /
equivalent), keys are typed to a single algorithm, and `aud`/`iss`/`exp` are
validated besides. This question is a proxy for "do you configure crypto
libraries deliberately or by default."

### Q10: Explain refresh token rotation and reuse detection. What attack does reuse detection address?

**Answer:** Every refresh is an exchange: present refresh token `R1`, get a
new access token plus `R2`, and `R1` is consumed — refresh tokens are
single-use. Reuse detection covers the theft case: if a consumed `R1` is
presented again, two parties hold it — the legitimate client and a thief —
and the server can't tell which one is asking now. The correct response is to
**revoke the entire token family** and force re-authentication, ending the
attacker's persistence. Without rotation, a stolen refresh token is silent,
long-lived access. Implementation notes that score points: store tokens
hashed with a family ID and device metadata, allow a few seconds' grace for
legitimate retry races, and alert on reuse events since each one is a
probable compromise signal.

### Q11: Walk through the authorization code + PKCE flow and explain what PKCE actually prevents.

**Answer:** Client redirects the browser to the authorization server with
`client_id`, exact `redirect_uri`, `state`, and a `code_challenge` =
SHA-256 of a random `code_verifier` the client keeps. The user authenticates
at the AS (the client never sees credentials), the AS redirects back with a
one-time code, and the client exchanges code + `code_verifier` on the back
channel; the AS checks the hash matches. PKCE prevents **stolen codes from
being useful**: the code transits the front channel (browser redirects,
logs, history, and on mobile, potentially a malicious app registered on the
same custom URL scheme), but exchanging it requires the verifier, which
never left the client and can't be derived from the challenge. `state`
separately prevents callback CSRF. OAuth 2.1 requires PKCE for *all* clients
— including confidential ones, where it also mitigates code injection.

### Q12: Why were the implicit and password grants removed, and what replaced them?

**Answer:** **Implicit** returned access tokens directly in the redirect URL
fragment — exposed to browser history, any JS on the page, and referrer
leakage — with no client authentication and no way to apply PKCE. It existed
only because pre-CORS browsers couldn't call the token endpoint from JS;
CORS removed the excuse. **ROPC (password grant)** has the client collect
the user's password, defeating OAuth's core purpose: the client sees the
credential, the IdP can't apply MFA or passkeys, and users get trained to
type passwords into arbitrary apps. Both are gone in OAuth 2.1. Replacement
for both: **authorization code + PKCE**, in the browser for SPAs and via a
system browser (`ASWebAuthenticationSession`/Custom Tabs) for native apps.

### Q13: A service accepts OIDC ID tokens as API credentials. What's wrong and what's the fix?

**Answer:** ID tokens and access tokens have different **audiences and
purposes**. An ID token's `aud` is the *client application* — it's proof of
authentication for the client to consume, not authorization to call an API.
If your API accepts ID tokens, any other application using the same IdP can
obtain an ID token for the same user (issued for *their* client ID) and call
your API with it — cross-client token replay. The fix: the API accepts only
**access tokens** whose `aud` matches the API's own identifier, validates
`iss`, signature (pinned algorithms via JWKS), `exp`, and then applies
scope and permission checks. The one-line takeaway interviewers want:
**"ID token = who the user is, for the client; access token = what may be
done, for the API — and `aud` validation is what enforces the difference."**

### Q14: Your service fetches user-configured webhook URLs. Design the SSRF defenses.

**Answer:** Layered, because each individual control has bypasses. In code:
enforce `https` and standard ports; resolve DNS yourself and reject
private/link-local/metadata ranges (RFC 1918, `127.0.0.0/8`,
`169.254.169.254`, `fc00::/7`), then **connect to the validated IP** rather
than re-resolving (kills DNS rebinding); disable redirect following or
re-validate every hop; apply timeouts and response-size caps. In
architecture — the control that survives code bugs: run the fetcher as a
dedicated egress service/proxy in a network segment with **no route to
internal services**, so even a bypass reaches nothing. On cloud: IMDSv2
required and a minimal instance role, so metadata access yields little.
Extras that signal depth: verify webhook deliveries with HMAC signatures so
receivers can authenticate you, and log/alert on blocked fetch attempts.

### Q15: How do you manage secrets across services and CI, and what's your response when one leaks into git?

**Answer:** Runtime injection from a secret store (Vault, AWS Secrets
Manager/SSM) with per-service IAM scoping; nothing in the repo or baked into
images. Better than managing secrets: **eliminating them** — cloud IAM roles
for service identity and OIDC federation for CI (GitHub Actions exchanging
its identity token for short-lived cloud creds) remove long-lived keys
entirely. Design rotation in from day one: two valid credentials during
rollover, no restart-the-world coupling. On a leak: **rotate immediately —
the secret is burned the moment it touches git**, because clones, forks, CI
caches, and scrapers mean history rewriting is cosmetic. Then audit usage
logs for the exposure window, add pre-commit/CI scanning (`gitleaks`), and
fix the path that let it happen. Candidates who say "rewrite history and
move on" fail this question.

---

## Staff

### Q16: Scenario — design authentication for a product with a web SPA, native mobile apps, and a public API. Walk through your choices.

**Answer:** Structure the answer by client, then unify. **Identity provider:**
don't build one — a managed IdP or hardened Keycloak/Ory; all clients use
**authorization code + PKCE** against it. **Web SPA:** either (a) tokens
handled via a thin backend-for-frontend that keeps them in `HttpOnly`
`SameSite` cookies — strongest against XSS token theft, my default — or (b)
access token in memory + refresh token in an `HttpOnly` cookie. Access
tokens 5–15 min; rotating single-use refresh tokens with reuse detection.
**Mobile:** same code+PKCE flow in a system browser sheet (enables passkeys
and SSO), tokens in Keychain/Keystore, longer-lived refresh tokens bound to
the device, revocable per device server-side. **Public API:** OAuth2 —
client credentials for machine integrations, code+PKCE for third parties
acting on behalf of users; scoped tokens, per-client rate limits. **Login
methods:** passkeys as the promoted default, password+TOTP as fallback,
recovery designed deliberately (recovery codes; support-driven reset with
verification) since it becomes the weakest link. **Cross-cutting:**
JWKS-published asymmetric signing with `kid` rotation, `aud` per API,
services validate locally at the gateway; token-family revocation on
compromise signals; auth events logged for detection. Close with the
trade-off you accepted: BFF adds a hop and a component but removes the
worst browser-side failure mode — that's the kind of explicit trade
statement staff interviews reward.

### Q17: Scenario — a pentest reports IDOR on several endpoints. Fixing the instances is easy; what do you do so this class of bug stops recurring?

**Answer:** Treat it as a systemic failure, not N bugs. **Architecture:**
move authorization from per-handler ad-hoc checks to a **centralized,
deny-by-default policy layer** (middleware + policy objects, or an engine
like OPA/Cedar for complex rules) so a forgotten check fails closed;
scope data access at the repository/query layer (`WHERE tenant_id = ?`
applied by construction — e.g. Postgres RLS or a query builder that demands
tenant context) so handlers *can't* fetch cross-tenant rows. **Verification:**
authorization test suite asserting user A gets 403/404 on user B's resources
for every resource type — cheap, high-ROI, regression-proof; add negative
tests to the definition of done for new endpoints. **Process:** threat-model
new features at design time (lightweight — "who can call this, on whose
data?"), security review triggers for endpoints touching new resource
types, and audit logging of authz denials to detect probing in prod.
**Secondary hardening:** non-guessable IDs (UUIDs) reduce discoverability
but are *not* the fix — the check is. What's being graded is exactly that
you fix the system that produced the bugs, and that you know obscurity isn't
authorization.

### Q18: Scenario — design service-to-service authentication for ~40 internal microservices, and explain how you'd roll it out without an outage.

**Answer:** **Goal:** every request carries a verifiable workload identity;
no shared static secrets. **Mechanism:** mTLS via a service mesh (or
SPIFFE/SPIRE) for transport-level workload identity with short-lived,
auto-rotated certs; plus, where request-level context matters, short-lived
JWTs from an internal token service (client-credentials style) carrying the
calling service's identity and scopes — many shops run both: mTLS for
"which workload," tokens for "on whose behalf" (user context propagated
end-to-end rather than trusting an internal header). **Authorization:**
an explicit service-to-service allow-graph — service A may call B's
`/charge`, not `*` — enforced at the mesh or a lightweight policy layer;
default deny. **Rollout without outage:** permissive/dual mode first
(accept both mTLS and legacy, log what *would* be denied), dashboard the
violations, migrate service by service, then flip to enforce per namespace
— never big-bang enforcement off observability you don't have. **Operational
realities that distinguish staff answers:** cert rotation must be fully
automated or it becomes the outage; the CA/token service is now
tier-0 infrastructure with its own HA story; break-glass paths and
bootstrap (how does the first credential get there — cloud IAM identity,
not a secret in config) must be designed, not improvised; and the audit
trail of who-called-what becomes your incident-response backbone.
