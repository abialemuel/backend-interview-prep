# Authentication and Sessions

How users prove who they are, and how the server remembers. This file covers
the mechanics interviewers probe first: sessions vs tokens, JWT sharp edges,
refresh rotation, password storage, MFA, passkeys, and SSO vocabulary.

---

## Session-based vs token-based auth

The fundamental split is **where the authoritative state lives**.

| | Server-side sessions | Self-contained tokens (JWT) |
|---|---|---|
| State | Server stores session record; client holds an opaque ID | Client holds signed claims; server stores nothing per session |
| Revocation | Trivial — delete the record | Hard — token is valid until expiry unless you add a denylist |
| Scaling | Needs shared store (Redis) across nodes | Any node validates with the public key; no shared read path |
| Cross-service | Every service must reach the session store | Any service holding the key can verify locally |
| Payload | Lookup returns anything | Claims baked in at issue time; stale until reissue |
| Typical transport | Cookie (`HttpOnly`, `Secure`, `SameSite`) | `Authorization: Bearer` header, or cookie |
| Size on the wire | ~32 bytes of ID | Hundreds of bytes to a few KB per request |

The honest interview framing: **sessions are the simpler, safer default for a
single web application**; tokens earn their complexity when many independent
services must verify identity without a shared synchronous dependency
(microservices, third-party APIs, mobile clients hitting multiple backends).

A session lookup against Redis is sub-millisecond — "sessions don't scale" is
rarely true at the scales most companies operate. The real JWT advantage is
**decoupling verification from a central store**, and the real cost is
**revocation and staleness**. Lead with that trade-off.

Hybrid is the common production answer: short-lived JWT access tokens
(5–15 min) for stateless verification, plus a server-side record for the
refresh token so logout and compromise response stay possible.

## JWT structure

A JWT is three base64url segments: `header.payload.signature`.

```json
// header
{ "alg": "ES256", "typ": "JWT", "kid": "2026-05-key-1" }

// payload (claims)
{
  "iss": "https://auth.example.com",
  "sub": "user-42",
  "aud": "api://orders",
  "exp": 1789000000,
  "iat": 1788999100,
  "jti": "b1c2...",
  "scope": "orders:read orders:write"
}
```

The signature covers header + payload. Two signing families:

- **HMAC (HS256)** — one shared secret signs *and* verifies. Every verifying
  service can also mint tokens. Acceptable only when issuer and verifier are
  the same trust domain.
- **Asymmetric (RS256, ES256, EdDSA)** — private key signs, public key
  verifies. Verifiers can't forge tokens. **Default to asymmetric** whenever
  more than one service verifies; publish keys via a JWKS endpoint and use
  `kid` for rotation.

!!! note "JWTs are signed, not encrypted"
    Anyone can base64-decode the payload. Never put secrets, PII you wouldn't
    show the user, or internal identifiers you consider sensitive into JWT
    claims. (JWE exists for encrypted tokens but is rare in app backends.)

## JWT pitfalls (the part interviewers actually care about)

### Algorithm confusion

The classic vulnerabilities:

- **`alg: none`** — early libraries honored a header claiming the token is
  unsigned. Any library from the last decade rejects this, but you must still
  configure an explicit allowlist.
- **RS256 → HS256 downgrade** — an attacker takes a token verified with an
  RSA *public* key, switches `alg` to HS256, and signs it using the public
  key bytes *as the HMAC secret*. If the library lets the token pick the
  algorithm, the attacker mints valid tokens from public material.

Defense: **the verifier chooses the algorithm, never the token.**

```go
// Go, golang-jwt: pin the expected method
token, err := jwt.Parse(raw, keyFunc,
    jwt.WithValidMethods([]string{"ES256"}),
    jwt.WithAudience("api://orders"),
    jwt.WithIssuer("https://auth.example.com"),
    jwt.WithExpirationRequired(),
)
```

Also always validate `exp`, `iss`, and `aud`. Skipping `aud` lets a token
issued for one API be replayed against another — a real and common bug.

### Storage on the client

- **Web:** `localStorage` is readable by any JS, so one XSS exfiltrates every
  user's token. Prefer an `HttpOnly; Secure; SameSite=Lax` cookie — XSS can
  still *use* the session while the page is open, but can't steal the
  credential for offline replay. Cookies reintroduce CSRF, so pair with
  `SameSite` plus CSRF tokens for state-changing routes. In-memory storage
  (a JS variable) is a reasonable SPA middle ground, paired with a
  cookie-carried refresh token.
- **Mobile:** OS keystore (iOS Keychain, Android Keystore) — never plain
  shared preferences or files.

### Revocation

The defining weakness: a stateless token is valid until `exp` no matter what
happened since — logout, password change, account compromise, permission
downgrade. Options, in rough order of preference:

1. **Short expiry + refresh tokens** — access token lives 5–15 minutes; all
   revocation decisions happen at the refresh endpoint, which *does* check
   server state. Compromise window shrinks to minutes.
2. **Denylist of revoked `jti`s** — a Redis set consulted on each request.
   Works, but you've reintroduced the shared store JWTs were meant to avoid;
   at that point ask whether sessions were the right answer.
3. **Per-user token version** — store `token_version` on the user row, embed
   it as a claim, bump to invalidate everything. One cheap indexed read per
   request.
4. **Key rotation** — nuclear option; logs out everyone.

If an interviewer asks "how do you log out a JWT," the senior answer is:
"individually, you mostly don't — you design so you don't need to, via short
expiry and stateful refresh."

## Refresh token rotation

Long-lived refresh tokens are high-value theft targets, so modern practice
(and OAuth 2.1) mandates **rotation with reuse detection**:

1. Client exchanges refresh token `R1` → receives access token + new `R2`;
   `R1` is marked used.
2. Every refresh repeats this — each refresh token is **single-use**.
3. **Reuse detection:** if `R1` arrives again, two parties hold it — the
   client and a thief. You can't tell which one is asking, so revoke the
   whole token family and force re-authentication.

```text
login  -> AT1 + R1
refresh(R1) -> AT2 + R2          R1 consumed
refresh(R1) again  -> REUSE DETECTED -> revoke family, force re-login
```

Store refresh tokens server-side **hashed** (SHA-256 is fine — they're
high-entropy random strings, not passwords), with family ID, expiry, and
device metadata. This also gives you a "sign out other devices" feature for
free. Allow a small grace window (a few seconds) for legitimate
network-retry races on the refresh call.

## Password storage

Passwords are stored as slow, salted hashes so a leaked database can't be
reversed at scale. Rules:

- **Use argon2id or bcrypt.** Argon2id is the current recommendation —
  memory-hard, so GPU/ASIC cracking rigs lose their advantage. Typical
  server-side parameters: 19–64 MiB memory, iterations tuned so hashing takes
  ~100–300 ms, parallelism 1–2. Bcrypt (cost 10–12) remains perfectly
  defensible and is the incumbent everywhere; scrypt sits in between. Fast
  hashes (MD5, SHA-1, SHA-256 — even salted) are disqualifying answers: an
  attacker with a dump tries billions per second.
- **Salts are per-user and automatic** in modern APIs (`password_hash`,
  `golang.org/x/crypto/argon2`, libsodium) — they defeat rainbow tables and
  make identical passwords hash differently. You never manage salts manually.
- **Peppers** (a server-side secret mixed in, e.g. HMAC the password before
  hashing) add defense when the DB leaks but the app host doesn't. Optional;
  bcrypt's 72-byte truncation makes pre-HMAC useful there anyway.
- **Upgrade on login.** Verify against the stored hash; if parameters are
  outdated, rehash with current settings while you have the plaintext.
- **Hashing is not the whole story.** Rate-limit and lock out failed
  attempts, check candidates against breached-password lists (the
  Pwned Passwords k-anonymity API), and require re-auth for sensitive
  actions. Length > composition rules: NIST guidance is long passphrases,
  no forced periodic rotation, no "one uppercase one symbol" theater.

## Multi-factor authentication

Something you know + something you have/are. Ranked by strength:

1. **Passkeys / security keys (FIDO2)** — phishing-resistant; the credential
   is origin-bound, so a fake login page gets nothing.
2. **TOTP apps** — shared secret at enrollment, 6-digit codes in 30 s
   windows. Solid, but phishable in real time by relay ("adversary in the
   middle") kits.
3. **Push approval** — phishable via prompt-bombing unless it uses number
   matching.
4. **SMS** — better than nothing, but SIM-swap attacks make it the weakest
   factor. Don't build new systems on it.

Backend implementation notes: store TOTP secrets encrypted, allow ±1 window
of clock drift, rate-limit verification attempts, issue single-use hashed
recovery codes at enrollment, and require a fresh MFA challenge before
disabling MFA or changing the enrolled factor (otherwise a stolen session
removes the second factor).

## Passkeys / WebAuthn

Passkeys are the industry's replacement for passwords, and increasingly a
standard interview topic. Mechanics:

- **Registration:** the authenticator (phone, laptop, security key) generates
  a key pair *per site*. The public key goes to your server; the private key
  never leaves the device (or syncs via the platform's encrypted keychain —
  iCloud Keychain, Google Password Manager).
- **Login:** server sends a random challenge; device signs it after local
  user verification (biometric/PIN); server verifies with the stored public
  key.

Why it matters — the properties you should be able to rattle off:

- **Nothing to leak:** the server stores only public keys. A database dump
  yields nothing crackable.
- **Phishing-resistant:** the browser scopes the credential to the origin
  (relying-party ID). A lookalike domain cannot request the real
  credential — this kills the attack class that MFA codes still fall to.
- **No shared secret in transit**, so no credential-stuffing or replay.

Server-side responsibilities: generate and persist single-use random
challenges, verify the origin and RP ID hash in the authenticator response,
check the signature counter where available (clone detection), and store
multiple credentials per user (people have several devices). Use a maintained
library (`go-webauthn`, `webauthn` for PHP/Node) — the CBOR/attestation
parsing is not something to hand-roll.

Practical rollout answer for interviews: passkeys ship **alongside** password
+ MFA as the preferred method, with account recovery being the new weakest
link — if recovery falls back to email alone, your auth is only as strong as
the user's inbox.

## SSO concepts

Single sign-on: authenticate once with a central **identity provider (IdP)**;
applications (**service providers / relying parties**) trust its assertions.

- **SAML 2.0** — XML-based, dominant in older enterprise software. The IdP
  posts a signed XML assertion to the SP. Verbose and historically prone to
  XML-signature-wrapping bugs, but "we support SAML" is still the enterprise
  sales checkbox.
- **OIDC** — the modern JSON/OAuth2-based equivalent (next file). New
  integrations default here.
- **Directory sync / SCIM** — provisioning and deprovisioning users
  automatically. The security payoff of SSO is that **offboarding happens in
  one place**: disable the IdP account and access to everything dies. Without
  SCIM or short sessions, ex-employees keep working sessions in every app.

Vocabulary worth having: IdP-initiated vs SP-initiated flows, JIT (just-in-
time) provisioning, and the fact that internally, SSO pairs with short
service sessions so IdP revocation actually propagates.

---

## Quick self-check

- Why is "JWTs scale better than sessions" an incomplete argument?
- Walk through the RS256→HS256 confusion attack and the one-line defense.
- What does refresh token reuse detection protect against, exactly?
- Why argon2id over SHA-256-with-salt? What property matters?
- What makes passkeys phishing-resistant when TOTP is not?
