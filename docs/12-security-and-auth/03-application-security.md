# Application Security

The defensive engineering layer: what you do in code, config, and process so
your service resists the attacks that actually happen. Interviewers use this
material for follow-ups — "OK, now how do you secure that endpoint?" — and
for judging whether security is a reflex or an afterthought in your work.

Organizing principle: **validate at the boundary, encode at the output,
authorize on every object, and give every component the least privilege it
needs.**

---

## OWASP Top 10, applied to backend work

The OWASP Top 10 is a vocabulary, not a checklist. The categories that
dominate backend interviews:

### Broken access control (#1 for a reason)

The most common real-world vulnerability class, and mostly *boring* bugs:

- **IDOR / missing object-level checks**: `GET /orders/991` verifies the
  user is logged in but not that order 991 is *theirs*. Authentication
  passed; authorization never ran.
- **Missing function-level checks**: the admin button is hidden in the UI,
  but `POST /admin/refunds` never verifies the role server-side.
- **Mass assignment**: binding request JSON straight onto a model lets a
  client send `"role": "admin"`. Use explicit allowlists/DTOs for writable
  fields.

Defenses: centralize authorization (middleware/policy layer) so checks can't
be forgotten per-handler; **deny by default**; always scope queries by owner
(`WHERE id = ? AND user_id = ?`); write tests that assert user A gets 403/404
on user B's resources — this is the highest-ROI security test you can write.

### Injection

Any pattern where user input is concatenated into an interpreter's
instructions: SQL, OS commands, LDAP, NoSQL operators, template engines —
and, in 2026 interviews, **prompt injection** in LLM-backed features follows
the same shape.

```go
// VULNERABLE — input becomes SQL text
db.Query("SELECT id FROM users WHERE email = '" + email + "'")

// SECURE — input stays data via parameterization
db.Query("SELECT id FROM users WHERE email = ?", email)
```

The universal principle: **keep data out of the instruction channel.**
Parameterized queries for SQL; `exec`-style APIs with argument arrays (never
`sh -c` with concatenation) for commands; allowlist-then-interpolate for the
rare identifier (column name, sort direction) that can't be parameterized.
Escaping is the fallback, parameterization the default.

### SSRF

Your server fetches a user-supplied URL (webhooks, importers, PDF renderers,
image proxies) and can be aimed at things only *it* can reach: cloud metadata
(`169.254.169.254` — credential theft; the Capital One breach), internal
admin panels, `localhost` services.

Naive URL blocklists fail to redirects, DNS rebinding, IPv6-mapped addresses,
and decimal-encoded IPs. Real defenses, layered:

- **Allowlist** destination hosts when the set is known.
- Resolve DNS yourself, validate the IP against private/link-local ranges
  (RFC 1918, `127.0.0.0/8`, `169.254.0.0/16`, `fc00::/7`), and **connect to
  the validated IP** — don't resolve twice.
- Disable/re-validate redirects; enforce scheme (`https`) and ports.
- **Network-layer egress control** — route user-driven fetches through a
  dedicated proxy in a segment with no route to internal services. The
  network control is the one that survives code bugs.
- On AWS: require IMDSv2 and keep instance roles minimal, so even a
  successful SSRF yields little.

### Insecure deserialization

Deserializing untrusted data with formats that can instantiate arbitrary
objects (PHP `unserialize`, Python `pickle`, Java native serialization,
YAML `load`) is remote-code-execution-adjacent: gadget chains in your
dependency tree turn "parse this blob" into "run this code."

Defense: **use data-only formats** (JSON, protobuf) mapped onto explicit
DTOs; if a native format is unavoidable, restrict allowed classes and
HMAC-sign the payload so only your own output is accepted. Same reasoning
applies to `pickle`-based caches and job queues fed by anything untrusted.

### The rest, in one pass

Worth recognizing on sight: **cryptographic failures** (plaintext PII, weak
hashing, homemade crypto — use TLS, argon2id, libsodium/KMS), **security
misconfiguration** (default credentials, debug endpoints in prod, public S3
buckets, verbose stack traces to clients), **vulnerable components**
(supply chain, below), **authn failures** (file 01), **logging/monitoring
failures** — you can't respond to what you can't see; log auth events and
authz denials with actor/action/target, and never log secrets or tokens.

## Input validation

- Validate **at the trust boundary** (the handler/controller), not deep in
  the stack; fail loudly with 4xx.
- **Allowlist over blocklist**: define what valid input *is* (type, length,
  range, format, enum membership) and reject everything else. Blocklists of
  known-bad strings always miss encodings you didn't think of.
- Parse into **typed values/DTOs** at the edge so the rest of the codebase
  never handles raw strings — "parse, don't validate twice."
- Validation is *not* output encoding: a name like `O'Brien <3` can be valid
  input and still needs parameterization in SQL and escaping in HTML.
  Different layers, both mandatory.
- Don't forget non-body inputs: headers, path params, file names,
  content-types, and sizes (reject oversized payloads before parsing).

## CORS, explained correctly

The most misexplained topic in interviews. CORS does not "secure your API" —
it is a **browser mechanism that selectively relaxes the same-origin
policy**, which by default stops JS on `evil.com` from *reading* responses
from `api.yours.com`.

Getting it right means knowing what it doesn't do:

- CORS **does not protect against non-browser clients**. curl and backend
  scripts ignore it entirely. Authorization must happen server-side
  regardless.
- CORS mostly restricts **reading responses**, not sending requests — simple
  requests (e.g. a form POST) still reach and execute on your server. CSRF
  protection is therefore a separate concern.
- **Preflights** (`OPTIONS`) happen for non-simple requests (custom headers
  like `Authorization`, JSON content-type, PUT/DELETE); the server's
  `Access-Control-Allow-*` response tells the browser whether to proceed.

Configuration rules: allow specific origins from an exact-match allowlist —
never reflect the incoming `Origin` header back while also setting
`Access-Control-Allow-Credentials: true`, which effectively hands any
website credentialed access to your API. `*` with credentials is forbidden
by the spec for exactly this reason.

## CSRF

A logged-in user's browser is tricked (form on an attacker page) into
sending a state-changing request to your site, riding the cookies the
browser attaches automatically. Applies to **cookie-authenticated**
endpoints; pure `Authorization`-header APIs are inherently immune (the
attacker's page can't set that header cross-origin without CORS approval).

Layered defense: `SameSite=Lax` (default-ish in modern browsers) or `Strict`
on session cookies; **anti-CSRF tokens** (synchronizer or signed
double-submit) verified with a constant-time compare on every state-changing
request; strictly no side effects on GET; `Origin` header checks and — for
modern browsers — `Sec-Fetch-Site` as defense in depth.

## Secrets management

- Secrets never live in the repo — not in code, not in committed `.env`
  files. Scan history with `gitleaks`/`trufflehog`; a secret that ever
  touched git is burned — rotate it, don't just delete the line.
- Inject at runtime from a secret store (Vault, AWS Secrets Manager, SSM
  Parameter Store) with access scoped per service via IAM.
- **Prefer identities over secrets** where the platform allows: IAM roles,
  workload identity, and OIDC federation for CI (GitHub Actions →
  short-lived cloud creds) eliminate whole classes of long-lived keys.
- Design for **rotation**: two valid keys during rollover, no restart-the-
  world dependencies. Assume any secret will eventually need emergency
  rotation and rehearse it.
- Keep secrets out of logs, error messages, crash reports, and URLs.

## TLS basics

What a backend engineer must actually know:

- TLS gives **confidentiality, integrity, and server authentication**; the
  client verifies the server's certificate chains to a trusted CA and
  matches the hostname. TLS 1.3 is the norm (faster handshake, legacy
  ciphers removed); 1.2 is the compatibility floor.
- **Everything over TLS**, including service-to-service traffic — "internal
  network" is not a trust boundary in a zero-trust posture. Service meshes
  make **mTLS** (client certs both ways, giving workload identity) the
  low-effort default internally.
- Certificates are automated now (ACME/Let's Encrypt, cloud-managed certs);
  manual expiry is a self-inflicted outage. HSTS
  (`Strict-Transport-Security`) pins browsers to HTTPS.
- Don't disable certificate verification "temporarily" in HTTP clients
  (`InsecureSkipVerify: true`) — it disables the entire guarantee and it
  always leaks into prod.

## Security headers

The high-value set for anything serving browsers:

| Header | What it does |
|---|---|
| `Strict-Transport-Security` | Forces HTTPS for future visits |
| `Content-Security-Policy` | Restricts script/resource sources; the big XSS blast-radius reducer (`script-src 'self' 'nonce-...'`) |
| `X-Content-Type-Options: nosniff` | Stops MIME-sniffing responses into executable types |
| `X-Frame-Options` / CSP `frame-ancestors` | Clickjacking defense |
| `Referrer-Policy` | Stops URLs (and their tokens/IDs) leaking cross-site |
| `Cache-Control: no-store` | On authenticated/personal responses — shared caches must not keep them |

Pure JSON APIs need less, but `nosniff`, correct `Content-Type`, and
`no-store` on sensitive responses still apply.

## Principle of least privilege

Every principal — human, service, container, CI job, DB account — gets the
minimum access required, so a compromise of one component is a contained
incident instead of a full breach.

In practice: per-service DB accounts with table-level grants (the API's
account can't `DROP TABLE`; the read-only reporting account can't write);
IAM roles scoped to specific resources/actions rather than `*`; containers
running as non-root with read-only filesystems and no unneeded capabilities;
network segmentation so services reach only declared dependencies; scoped
short-lived CI credentials; and time-bound, audited human access to
production instead of standing admin. Least privilege is also a *detection*
tool — when the web tier suddenly attempts `iam:CreateUser`, the denial is
your alarm.

## Supply-chain awareness

Most of your codebase is other people's code, so it's part of your attack
surface:

- **Lockfiles** (`go.sum`, `composer.lock`, `package-lock.json`) — commit
  them; they pin exact versions and hashes so builds are reproducible and a
  hijacked minor release doesn't auto-flow into prod.
- **Dependency scanning in CI** — `composer audit`, `npm audit`,
  `govulncheck` (which checks whether the vulnerable *function* is actually
  reachable), Dependabot/Renovate for automated patch PRs. Alert fatigue is
  real: prioritize by reachability and exposure.
- **Know the attack shapes**: typosquatting, **dependency confusion**
  (a public package shadowing your internal name — reserve names, pin
  registries), maintainer-account takeover, and malicious install scripts.
  The `xz` backdoor made "who maintains this?" a mainstream question.
- Heavier-duty controls to name-drop accurately: SBOMs, artifact signing and
  provenance (Sigstore/SLSA), and building from an internal proxy/mirror
  rather than straight from public registries.

---

## Quick self-check

- What's the difference between authentication and authorization failure in
  an IDOR bug? Where's the fix?
- Your service fetches user-provided webhook URLs. List three layered SSRF
  defenses, including the non-code one.
- "We're safe from CSRF because we have CORS configured" — what's wrong with
  that sentence?
- A secret was committed and force-pushed away. What's the actual remediation?
- Why do lockfiles matter for security and not just reproducibility?
