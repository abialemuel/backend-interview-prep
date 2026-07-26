# OAuth2 and OIDC

OAuth2 is **delegated authorization** — letting a client act against an API
on someone's behalf without holding their password. OIDC is the
**authentication layer** built on top of it. Interviewers probe whether you
know which grant to use when, why the deprecated ones died, and the
difference between an ID token and an access token. Precision with the
vocabulary is most of the battle.

---

## The four roles

| Role | Who it is | Example |
|---|---|---|
| **Resource owner** | The user who owns the data | You, with a Google account |
| **Client** | The app requesting access | A scheduling app that reads your calendar |
| **Authorization server (AS)** | Issues tokens after authenticating the owner | Google's OAuth endpoints |
| **Resource server (RS)** | The API that accepts access tokens | Google Calendar API |

Two client types with different security properties:

- **Confidential clients** can keep a secret — server-side web apps, backend
  services. They authenticate to the AS with a client secret (or better,
  private-key JWT / mTLS).
- **Public clients** cannot — SPAs, mobile apps, CLIs. Anything shipped to
  the user's device is decompilable, so a "secret" embedded there is not one.
  Public clients rely on PKCE and redirect-URI binding instead.

## Grant types you should know

### Authorization code + PKCE — the default for anything user-facing

The flow every "log in with X" and every first-party mobile/web login should
use:

```text
1. Client -> browser redirect to AS /authorize
     ?response_type=code&client_id=...&redirect_uri=...
     &scope=openid calendar:read&state=<random>
     &code_challenge=BASE64URL(SHA256(verifier))&code_challenge_method=S256
2. User authenticates at the AS (password, passkey, SSO — client never sees it)
3. AS redirects back: ?code=<one-time-code>&state=<same random>
4. Client (back channel) -> POST /token
     code + redirect_uri + code_verifier (+ client secret if confidential)
5. AS verifies SHA256(verifier) == stored challenge -> issues tokens
```

**PKCE** (Proof Key for Code Exchange, "pixy") is the piece to explain well.
The authorization code travels through the browser front channel — visible in
redirects, logs, referrers. Without PKCE, anyone who steals the code can
exchange it. With PKCE, exchanging the code requires the `code_verifier` — a
random secret that never left the client and can't be derived from the
`S256` challenge. PKCE was invented for mobile apps (malicious apps
registering the same custom URL scheme could intercept codes) but is now
required for **all** clients, confidential included, because it also
mitigates code injection.

`state` is a separate, older control: a random value the client checks on
return, preventing CSRF on the callback (an attacker binding *their* code to
*your* session). With PKCE it's partially redundant but still standard.

### Client credentials — machine to machine

No user involved: a backend service authenticates as itself
(`grant_type=client_credentials`) and gets a token scoped to its own
permissions. This is the standard pattern for service-to-service auth,
cron jobs, and internal APIs. The trade-off vs plain API keys: short-lived
tokens with scopes and central rotation, at the cost of running/renting an AS.

### Device authorization grant — no browser on the device

For TVs, CLIs, IoT: the device shows a short user code and URL
(`example.com/activate`, code `WDJB-MJHT`), the user approves on their phone,
and the device polls the token endpoint until approval. This is what
`gh auth login` and every smart-TV app do.

### Refresh token grant

Exchanges a refresh token for a new access token — mechanics and rotation
covered in `01-authentication-and-sessions.md`. For public clients, rotation
with reuse detection is mandatory in current practice.

## The dead grants — and why (a favorite interview question)

- **Implicit grant** (`response_type=token`): access tokens were returned
  directly in the redirect **URL fragment** — exposed to browser history,
  JS on the page, and anything reading the location bar, with no client
  authentication and no PKCE possible. It existed only because pre-CORS
  browsers couldn't make the back-channel token call. CORS fixed that;
  implicit is deprecated and removed in OAuth 2.1. SPAs use authorization
  code + PKCE.
- **Resource owner password credentials (ROPC)**: the client collects the
  user's password and posts it to the AS. This defeats the entire point of
  OAuth — the client sees the password, MFA and passkeys can't participate,
  phishing training ("only type your password on the IdP's page") is
  undermined. Also removed in OAuth 2.1. Its one historical use (trusted
  first-party apps before better options) is now served by code + PKCE in a
  system browser / `ASWebAuthenticationSession`.

## OIDC on top

OAuth2 alone answers "may this client call this API?" — it deliberately says
nothing about *who the user is*. Pre-OIDC, everyone improvised login on top
of OAuth by calling some `/me` endpoint, each vendor differently and often
insecurely. **OpenID Connect** standardizes authentication on the same rails:

- Add the `openid` scope (plus `profile`, `email` as needed) to an
  authorization code flow.
- The token response additionally contains an **ID token** — a JWT with
  claims about the authentication event: `iss`, `sub` (stable user ID),
  `aud` (your client ID), `exp`, `iat`, `nonce`, `auth_time`, and profile
  claims.
- Discovery: `https://issuer/.well-known/openid-configuration` publishes all
  endpoints and the **JWKS** (public signing keys) URI, so integration is
  mostly configuration.
- The `nonce` binds the ID token to the client session (replay protection),
  echoing the value the client sent in the authorize request.

### ID token vs access token — the distinction interviewers test

| | ID token | Access token |
|---|---|---|
| Audience | The **client** (`aud` = client ID) | The **resource server** (`aud` = API) |
| Purpose | Proof the user authenticated; identity claims | Authorization to call an API |
| Format | Always a JWT (spec-defined) | Opaque or JWT — AS's choice |
| Correct use | Consumed by the client to establish its own session | Sent as `Authorization: Bearer` to the RS |
| Wrong use | Sending it to an API as credentials | Introspecting it in the client for identity |

The classic integration bug: an API accepting **ID tokens** as API
credentials. Any other client of the same IdP can obtain an ID token for the
same user and call your API with it — the `aud` check would have caught this,
which is exactly why validating `aud` is non-negotiable.

## Token validation on the resource server

For JWT access tokens, on every request:

1. Fetch and cache signing keys from the JWKS endpoint; select by `kid`.
2. Verify the signature with a **pinned algorithm allowlist**.
3. Validate `iss` (exact match), `aud` (your API's identifier), `exp`/`nbf`
   (small clock skew tolerance), and then apply scope/permission checks.

```php
// Concept (firebase/php-jwt style)
$claims = JWT::decode($token, JWK::parseKeySet($jwks), ['ES256']);
if ($claims->iss !== 'https://auth.example.com') throw new Unauthorized();
if (!in_array('api://orders', (array)$claims->aud)) throw new Unauthorized();
// then: does 'scope' contain what this endpoint needs?
```

For **opaque** tokens, call the AS's **introspection endpoint**
(RFC 7662) — a per-request network hop, usually cached briefly. Trade-off:
opaque tokens are revocable instantly and leak nothing; JWTs verify locally
but carry the revocation problem. Many platforms use opaque tokens at the
edge and exchange them for internal JWTs at the gateway.

## Scopes vs permissions

A subtle senior-level distinction:

- **Scopes** express what the user **delegated to the client**:
  "this app may read my calendar" (`calendar:read`).
- **Permissions/roles** express what the **user themselves** may do in the
  system: "Alice is an admin of workspace 7."

A request must pass **both**: token has the scope AND the user has the
underlying permission. Scopes are coarse (per-API-area); real authorization —
especially **object-level** checks ("is Alice allowed to see order 991?") —
lives in your application or a policy layer, never in scopes alone. Encoding
fine-grained permissions into tokens also goes stale: a token minted before a
role downgrade still carries the old claims until expiry. Keep tokens coarse
and check fresh state for sensitive decisions.

## Common integration mistakes

- **Redirect URI validation that isn't exact-match.** Prefix or wildcard
  matching lets attackers receive codes at `evil.example.com` via crafted
  paths or open redirects on your domain. Register exact URIs.
- **Skipping `state`/`nonce`/PKCE** because "it works without them." Each
  removes a distinct attack (callback CSRF, ID token replay, code theft).
- **Accepting ID tokens at the API** (above).
- **Not validating `aud`/`iss`** — cross-service token replay.
- **Trusting unverified `email` claims for account linking.** An IdP that
  lets users set arbitrary unverified emails lets an attacker link into a
  victim's account. Link on `iss`+`sub`, never on email alone; treat
  email-match linking as a risk decision requiring `email_verified`.
- **Long-lived access tokens** compensating for a missing refresh flow —
  minutes, not days.
- **Rolling your own AS casually.** Building a compliant, secure AS is a
  product-sized job; default to a managed IdP (Auth0/Okta, Cognito, Entra
  ID) or a hardened open-source one (Keycloak, Ory, ZITADEL) and spend your
  effort on integration correctness.

## OAuth 2.1 direction

OAuth 2.1 consolidates a decade of best-current-practice into the spec, so
"what changes in 2.1?" doubles as "what is modern practice?":

- **PKCE required** for all authorization code flows.
- **Implicit and ROPC grants removed.**
- **Exact-string redirect URI matching** required.
- **Refresh tokens** for public clients must be sender-constrained or
  one-time-use (rotation).
- Bearer tokens banned from URL query strings.

Adjacent current topics worth name-dropping accurately: **sender-constrained
tokens** (DPoP, mTLS-bound tokens) that bind a token to a client key so a
stolen token can't be replayed; **PAR** (pushed authorization requests) and
**RAR** (rich authorization requests) in higher-assurance profiles like
FAPI (open banking).

---

## Quick self-check

- Explain PKCE to a junior in three sentences, including what attack it stops.
- Why exactly was the implicit grant deprecated? What replaced it for SPAs?
- A partner asks you to accept their ID token to call your API. What do you say?
- Opaque access tokens vs JWT access tokens — name both sides of the trade.
- Where do object-level permission checks belong, and why not in scopes?
