# [READY-PLAN] Yellow Card Treasury Portal — Unauth SSO-Lookup Oracle & Auth Surface

> **Status:** READY-PLAN — partially verified live 2026-09-03 from a datacenter IP, then
> **blocked by Cloudflare Turnstile + datacenter-IP rate limiting (429 code 1015).**
> NOT yet exploited to a confirmed differential and NOT submitted.
> To finish: run the exact recipe below from a **non-flagged residential IP + real browser**
> (solve the Turnstile) — this is the remaining step before submission.

**Program:** Yellow Card Financial bug bounty (confirmed live policy, 2026-06-08).
- Scope: Treasury portal `/` + **unauthenticated surface only** — "login page, password reset
  flow, public-facing HTTP endpoints" (https://yellowcard.io/blog/yellow-card-bug-bounty-program)
- Rewards: Level 0 Info … Level 4 Critical, flexible, based on severity.
- Interested in: customer data leaks, full system compromise, **bypassing auth/authz** → significant
  financial/reputational impact.
- Report to: `bugbounty@yellowcard.io` (include name, contact, discovery dt, URL, vuln type,
  step-by-step reproduction, screenshots/video).

**Asset nuclei (in-scope, all reachable):**
- `portal.yellowcard.io` — Vue SPA (Vercel/Cloudflare), HTTP 200, strict CSP.
- `api.yellowcard.io` — AWS API Gateway behind Cloudflare+Turnstile (this is where the bug lives).
- `sso.portal.yellowcard.io` — SSO endpoint (404 on bare `/`; Cloudflare `cf-ray`).

**CSP `connect-src` leak (source of the topology):**
`sso.portal.yellowcard.io`, `api.yellowcard.io`, `*.yellowcard.engineering`,
`wss://*.pusher.com`, `s3.eu-west-2.amazonaws.com` + `*.s3.eu-west-2.amazonaws.com`,
`*.usepylon.com`, Datadog browser intake.

---

## Finding 1 (primary) — Unauthenticated SSO / user-enumeration oracle: `GET /auth/sso-lookup`

### What the SPA does (recovered from `app.js`, Vue bundle)
The login flow calls, before auth:
```
GET /auth/sso-lookup?email={email}&turnstileToken={TURNSTILE}
GET /auth/sso-lookup?iss={issuer}&turnstileToken={TURNSTILE}
```
- If the submitted identity is an **SSO user** the app routes to an IdP (OAuth `/oauth2/authorize`);
  if a **password user** it shows the password form.
- Parameters are `email` + `turnstileToken`, OR `iss` + `turnstileToken`. Requires a valid
  Cloudflare **Turnstile** token (obtainable free in-browser).
- Response also feeds `GetSsoConfig` (`GET sso/config`, token-gated) and routes further auth
  states: `AUTH_BLOCKED`, `AUTH_EXPIRED`, `AUTH_INVALID`, `MFA_REQUIRED`, `MFA_SETUP`,
  `NEW_PASSWORD_REQUIRED`, `OTP_*`, `InvalidLogin`.

### Why it matters (SSO/account enumeration on the in-scope login flow)
The `email`-vs-`iss` branch and the differing response for a **registered SSO user vs unknown email**
form a **user/SSO-tenant enumeration oracle**. If the response distinguishes:
- valid email (routes to IdP / returns a token) vs invalid email (no IdP / different status),
then an attacker enumerates which accounts use SSO, and for SSO tenants can identify the
**identity provider** (via `iss` → IdP discovery) → enables targeted, authentic-looking
**SSO phishing / password-reset-watering** and confirms account existence (which then feeds
credential-stuffing and password-reset targeting).
This directly matches the program's "bypassing authentication" + "customer data" interests.

### Exact recipe to run from a residential IP + real browser (the remaining validation)
1. Load `https://portal.yellowcard.io/` in a real browser (residential IP) so `cf-mitigated`/
   Turnstile cookies + a valid `turnstileToken` are issued.
2. Open DevTools → Network. Submit `nonexistent-ok@example.invalid` in the login email field;
   capture the outbound `sso-lookup` request (method, query, headers).
3. Replay the SAME `sso-lookup` with the captured `turnstileToken`: again with an unknown email,
   then with a **known/likely account email** (e.g. `admin@yellowcard.io`, a test account you create
   under YOUR OWN mailbox, never a real customer).
4. Diff the three responses: status code + body (is `sso`/`iss`/`idp` present? status `true`?).
   A **200 with SSO/`iss` config for your own valid account vs `404`/different body for the invalid
   email** = confirmed enumeration oracle.

### Validation bar (when you can run it)
- Must show a **real differentially-observable difference** across valid(pass)/valid(sso)/invalid
  email — not just the presence of `/auth/sso-lookup`.
- If only your OWN test accounts are used and no real customer email is probed, you stay fully
  compliant with "unauth surface only + do not attempt to view others' data."
- Severity if confirmed: **Low–Medium** (SSO/user enumeration, auth-routing disclosure). Rises if
  `iss` lets you enumerate the **specific IdP tenant** per email (SSO-tenant enumeration).

---

## Finding 2 (verified, low value alone) — Unauth `POST /auth/encryption` returns CSRF token
- `POST https://api.yellowcard.io/auth/encryption` with `{}` + `Content-Type: application/json`
  → **HTTP 200** `{"csrf":"<32B base64>"}` **unauthenticated**, on AWS API Gateway
  (`x-amz-apigw-id`, `x-amzn-requestid`). Re-verified 2026-09-03 before rate-limit kicked in.
- This is the app's own client-side CSRF bootstrap for the login flow; on its own it is **Info**
  and **DO NOT SUBMIT standalone** (CSRF tokens issued for the login form are expected). Mention
  it only as context that the API is fully unauthenticated at `/auth/*` bootstrap.

---

## Confirmed infra/topology observations (context, not standalone findings)
- `api.yellowcard.io` is **Cloudflare + Turnstile + AWS API Gateway**; datacenter IP got
  `429 error code: 1015` (rate limited) after a handful of reads → treat as scanner/DDos parity
  note only, **not** a finding.
- API surface present in bundle (all downstream of the /auth/ bootstrap): `/auth/callback`,
  `/auth/encryption`, `/auth/sso-lookup`, `/portal/documents`, `/portal/transactions`,
  `/oauth2/authorize`, `/oauth2/token`, `sso/config`, OAuth/MFA/OTP state machine.

---

## Submission checklist (when validation completes)
- [ ] Run recipe from residential IP + real browser; capture burp/network traces with cookies redacted.
- [ ] Confirm differential across valid-pass / valid-sso / invalid email using ONLY your own test accounts.
- [ ] Severity + CVSS (expect Low if plain enum; Medium if SSO-tenant `iss` per-email disclosure).
- [ ] Screenshot the login flow routing for an SSO account vs invalid.
- [ ] Send to `bugbounty@yellowcard.io` with the checklist items from their policy.
- [ ] Do NOT probe real customer emails; halt at any authenticated/business-account data per policy.

## Blockers to completion
- Cloudflare **Turnstile** requirement → cannot be satisfied from datacenter IP; needs real
  browser + residential IP.
- **Rate limiting `429 1015`** on `api.yellowcard.io` from datacenter IP (permanent until IP
  rotates / egress changes).
- No account was created (program says `no_account_creation` guardrails for unauth-only scope);
  a **test account created by the researcher on their own mailbox** is the sanctioned way to get
  a known "valid" identity for the differential.
