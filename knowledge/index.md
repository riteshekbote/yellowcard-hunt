# Yellow Card hunt KB — verified learnings (RAG for all models)
> CRITICAL: testing is UNAUTHENTICATED-SURFACE ONLY on portal.yellowcard.io (login,
> password reset, public HTTP endpoints, TLS/headers). HALT + report immediately on
> any authenticated session or account data. Yellow Card is a crypto/stablecoin
> fintech (Africa) — treasury portal is high-value financial target.
> What pays: customer data leaks, full compromise, business-logic bypass with
> financial impact, auth bypass, major operational failure (non-DoS).

## REJECTED CLASSES (policy — do not propose)
- REJECTED brute force / rate limiting / lockout policies @ *: explicitly excluded.
- REJECTED missing HTTP security headers, cookie issues, SSL issues @ *: excluded.
- REJECTED clickjacking w/o exploit, mail config, known-vuln libraries, 3rd-party apps.
- REJECTED automated-scanning-only reports: must include PoC + detailed repro.
- REJECTED descriptive errors/headers, robots.txt/known public files.

## ALIVE SURFACE FACTS (verified)
- 2026-08-16 policy page: program active. Portal = portal.yellowcard.io (Treasury
  portal). Related assets: docs.yellowcard.engineering, help.yellowcard.io,
  portal.yellowcard.io/login. Contact bugbounty@yellowcard.io. (setup seed, UNVERIFIED
  live status of each host)
- 2026-08-16 (setup seed) KB: Yellow Card = stablecoin/digital-asset infrastructure
  (API suite + Treasury portal + fiat payment infra). High-value classes = business
  logic on treasury/funding flows, auth bypass, account-level IDOR.

## OPEN QUESTIONS
- Treasury portal stack/framework (to be fingerprinted on first probe)
- Which auth provider / password-reset implementation (reset flow is in scope!)
- Any public API (docs.yellowcard.engineering) exposing unauth endpoints

## FINDING INBOX (validated = move to reports/)
- (empty)
