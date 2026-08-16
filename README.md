# yellowcard-hunt

24/7 multi-model bug-hunting automation for the **Yellow Card Bug Bounty** program.

- **Scope**: `portal.yellowcard.io` (Treasury portal) — **UNAUTHENTICATED SURFACE ONLY**
- **Disclosure**: email `bugbounty@yellowcard.io` (policy: yellowcard.io/blog/yellow-card-bug-bounty-program)
- 5 opencode models hunt in parallel every 10 min; subdomain recon daily; JS recon every 5 min
- **Hard rule**: stop + report immediately if you obtain any authenticated session or business account data

## Yellow Card exclusions (no reward)
Brute force, rate limiting/lockout, missing headers, cookie/SSL issues, clickjacking w/o exploit,
mail config, known-vuln libs, automated-scanning-only reports, descriptive errors, robots.txt.
See `scope.yml`.

## What pays
Customer data leaks, full system compromise, business-logic bypass with financial/reputational
impact, auth/authorization bypass, major operational failure (non-DoS). Reward flexible by severity
(Info → Critical).

## Reporting
Email `bugbounty@yellowcard.io` with: name+contact, discovery date, URL, vuln type+description,
step-by-step repro + PoC, screenshots/video. Confidentiality + safe harbour apply; first clear
reproducible report wins.
