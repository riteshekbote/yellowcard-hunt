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
- 2026-08-16 FINDING (INFO/LOW candidate) @ POST https://api.yellowcard.io/auth/encryption — UNAUTHENTICATED encryption-init endpoint returns CSRF token: {"csrf":"brsc+W4jW/6Tal4m8FG4omsVj5O2bZFjcfYybmW9uzM="} (base64, ~64B). App axios.create({baseURL:"https://api.yellowcard.io/auth/encryption", headers:{"deviceid":<uuid>}}).post("") then uses token. ONLY api path reachable without Cloudflare block; all other paths → 403 "Bad request" (CF bot-fight). api.yellowcard.io/auth/encryption also gave {"message":"Missing Authentication Token"} on raw GET earlier — POST+deviceid passes.
- 2026-08-16 API SURFACE MAP recovered from portal.yellowcard.io/js/app.6d08659e.js (1.3MB, source-mapped with inputSourceMap!): base https://api.yellowcard.io with axios api token; methods recovered: users/login, users/refresh, users/password (PUT), users/verify-token, users/challenge-response, user/otp (PUT), user/reset-password, users/confirm-password-reset, users/add (invite), users/update (roles), users/resend-invite; partner/{accounts,users,apikey,rates,wallet,ip-address,ip-addresses,payout-config,profile,address,addresses,wallets,webhook,travel-rule,collection,disbursement,fiat-wallets,fiat-wallet,accept-agreements,low-balance-limit,settlement/withdrawal}; portal/{rates-and-limits,rates-and-limits/conversion,presigned-url,documents,delete-document,view-document,transactions,deposit-banks,partner-currencies/list}; commercial/client, commercial-entity, commercial-rates/portal, platform-rates/portal, report-jobs, report-jobs/create, sso/{config,configure,test,activate,delete,PATCH config}, channels, networks, logout, data (POST), transactions (POST), transaction/update (PUT). Financial crown jewels: partner/settlement/withdrawal (ProcessSettlement), partner/accounts (GetAccount), partner/rates (GetPartnerRates), portal/transactions (getAllPortalTransactions) — all auth-gated (need token).
- 2026-08-16 CLOUDFLARE GATE: api.yellowcard.io behind Cloudflare bot-fight; paths other than /auth/encryption → 403 "Bad request"; POST register → 429 code 1015 (CF). SPA shell (portal.yellowcard.io/*) always 200 same HTML. docs.yellowcard.engineering = GitBook-style (200, useReactApp class). policy portal login 200.
- 2026-08-16 (REJECTED-CLASS NOTE) brute force on login/forgot = excluded by policy; rate limiting excluded.
