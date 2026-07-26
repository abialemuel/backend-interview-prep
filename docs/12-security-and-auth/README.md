# Security & Auth — Backend Engineer Interview Prep

This section covers authentication, authorization, and application security
from the perspective of a backend engineer **building and defending** systems.
The emphasis is on decisions you actually make on the job: how users prove who
they are, how services prove who they are to each other, and how you keep the
whole thing from becoming next quarter's incident review.

## Why this section exists

Auth and security questions appear in **every backend interview loop**, at
every level, for three reasons:

1. **They're unavoidable in the work.** Every backend service authenticates
   something — users, other services, webhooks, CI pipelines. There is no
   "I've never touched auth" backend career.
2. **They're a cheap seniority probe.** A junior can recite "JWTs are signed
   tokens"; a senior can explain why revocation is the hard part and when a
   plain session cookie beats a token architecture; a staff engineer can
   design the auth story for a company (SSO, service-to-service identity,
   secrets rotation) and defend the trade-offs. Interviewers get a strong
   level signal from one question.
3. **Mistakes are asymmetric.** A slow query costs latency; a broken access
   control check costs the company. Interviewers want evidence you build with
   a security mindset by default, not as an afterthought.

The recurring mental model throughout these notes:

- **Authentication (authn)** — who are you? (login, tokens, sessions, MFA)
- **Authorization (authz)** — what are you allowed to do? (scopes, roles,
  permissions, object-level checks)
- **Defense in depth** — no single control is trusted to be perfect; layer
  validation, least privilege, network controls, and monitoring so one
  failure doesn't equal one breach.

Almost every interview answer in this domain is a **trade-off answer**:
stateless vs revocable, convenience vs blast radius, standard protocol vs
bespoke simplicity. Practice naming the trade-off explicitly.

## Files in this section

| File | Contents |
|---|---|
| `01-authentication-and-sessions.md` | Session vs token auth, JWT structure and pitfalls (alg confusion, storage, revocation), refresh token rotation, password storage (argon2id/bcrypt), MFA, passkeys/WebAuthn, SSO concepts. |
| `02-oauth2-and-oidc.md` | OAuth2 roles and grant types (authorization code + PKCE, client credentials, device flow), why implicit and password grants are dead, OIDC on top, token validation, scopes vs permissions, common integration mistakes, OAuth 2.1. |
| `03-application-security.md` | OWASP Top 10 applied to backend work (injection, broken access control, SSRF, insecure deserialization), input validation, CORS, CSRF, secrets management, TLS, security headers, least privilege, supply-chain hygiene. |
| `04-interview-questions.md` | 18 graded interview questions (junior → senior → staff) with model answers, including full design scenarios. |

## Recommended reading order

1. `01-authentication-and-sessions.md` — the foundation. Sessions, tokens,
   and password storage come up in the first ten minutes of any auth
   discussion, and everything in file 02 builds on the token vocabulary here.
2. `02-oauth2-and-oidc.md` — the standard protocols layered on top. Most
   real-world "add login to our app" and "integrate with a third party" work
   is OAuth2/OIDC, and interviewers increasingly expect precise vocabulary
   (grant types, PKCE, ID token vs access token).
3. `03-application-security.md` — the defensive engineering layer around
   everything else. This is where "how would you secure this endpoint?"
   follow-up questions live.
4. `04-interview-questions.md` — self-test after the others. The scenario
   questions (design auth for mobile + web, JWT vs sessions) are the ones
   most worth rehearsing out loud.

## How to use this material

- **Defend a choice, don't recite a list.** "JWT vs session" is not a trivia
  question — the interviewer wants you to pick one for a concrete context and
  defend it. Practice committing to an answer.
- **Know the two or three numbers that matter.** Access token lifetime
  (minutes, not days), bcrypt cost (~10–12, tuned to ~100–300 ms), argon2id
  memory (tens of MiB minimum). Precision signals experience.
- **Name the failure mode.** For every mechanism, know how it breaks: JWTs
  break at revocation, CORS breaks when treated as an auth control, API keys
  break at rotation. "Here's the sharp edge" answers score far above feature
  recitation.
- **Stay current.** Passkeys/WebAuthn and OAuth 2.1 direction are now
  standard interview material, not exotic extras. Content here reflects
  practice as of mid-2026; verify specifics (algorithm parameters, browser
  behavior) against current references when it matters.

> Conventions: examples use Go, PHP, and pseudo-HTTP where a concrete snippet
> helps; the concepts are language-agnostic. Everything here is written from
> the builder's/defender's perspective — the goal is designing systems that
> resist attack, and understanding attacks only deeply enough to defend
> against them.
