# [RESEARCH/HYPOTHESIS] Yellow Card API — Cross-Tenant IDOR by `sequenceId` (sender PII)

> **Type:** Research-only hypothesis, NOT yet validated (needs valid sandbox API keys).
> **Status:** BLOCKED ON CREDENTIALS. Every reachable endpoint is HMAC auth-gated (verified live
> 2026-09-03: all `/business/*` sub-resources return `401 AuthenticationError`). The unauth-only
> program scope forbids obtaining a key. This document fixes the exact test recipe so it can be
> run the moment valid sandbox credentials + HMAC secret are available.
> **Do NOT submit as a finding yet** — no exploit demonstrated.

---

## Why this is the highest-value lead on this API

`lookup-payment-by-sequenceid` returns, for a **client-supplied `sequenceId`**, the full
sender + destination PII of a payment. The `sequenceId` is **caller-chosen** (the docs describe it
as "a unique id for the transaction from your end" / "the identifier of the specific channel").
If the lookup does not scope results to the authenticated partner/tenant, a partner can retrieve
**another tenant's** payment record — a cross-tenant **IDOR / BOLA** disclosing:

- Sender: `name`, `email`, `phone`, `address`, `dob`, **`idNumber` + `idType` (e.g. passport)**
- Destination: `accountName`, `accountNumber`, `networkId`
- Financial: `amount`, `currency`, `rate`, `convertedAmount`, `status`, timestamps

This is precisely the "customer data leak" class the Yellow Card program says it wants.

## Exact documented response shape (from OpenAPI in `reference/lookup-payment-by-sequenceid.md`)
```
GET https://{api}.yellowcard.io/business/payments/{sequenceId}   (HMAC-authenticated)
      { "sender": { "name","country","phone","address","dob","email","idNumber","idType" },
        "destination": { "accountName","accountNumber","accountType","networkId" },
        "channelId","sequenceId","amount","currency","country","reason","partnerId","id",
        "status","sessionId","convertedAmount","rate","expiresAt","createdAt","updatedAt" }
```

## Auth model (from `docs/authentication-api.md`, verified in OpenAPI)
- `YcHmacV1 {apikey}:{signature}` + `X-YC-Timestamp` (ISO8601), message = `timestamp + path + method + base64(sha256(body))`.
- **Production additionally requires source-IP whitelisting**; sandbox does not.
- **Sandbox is NOT Cloudflare-blocked** (returns app-layer JSON) → safe place to validate.

## Two concrete weakness hypotheses to test with valid keys

### H1 (primary) — Cross-tenant IDOR via `sequenceId`
Modern routing: attacker partner **A** (own sandbox account, valid HMAC) tries
`GET /business/payments/{sequenceId}` where `sequenceId` is **B's** client-chosen value (guessed /
observed / leaked via a ref/URL / sequentially-derivable).
**Pass if:** A's request returns B's payment + sender PII (HTTP 200 with full payload) instead of
`403`/empty/scoped-to-tenant error. → **High/Critical IDOR-BOLA, PII.**
**Assumption to verify first:** whether `sequenceId` is (a) truly client-chosen and predictable, vs
(b) server-minted; docs say caller-chosen ("from your end") → strong prior it is enumerable.

### H2 (secondary) — HMAC replay / timestamp window
The scheme signs `timestamp + path + method + bodyhash` but the docs do not describe a
**nonce** or a **timestamp-tolerance window**. If `X-YC-Timestamp` is not bounded,
a captured request can be replayed indefinitely (idempotency/replay on
`submit-payment`, `submit-collection-request`, RFQ acceptance).
**Pass if:** replaying a captured sandbox `submit-*` request after its documented expiry
(`expiresAt`) is still accepted / double-executes. → **Medium**, financial-replay.

### H3 (secondary) — `resolve-bank-account` account-name enumeration
`POST` with `accountNumber` + network returns **`accountName` + `accountBank`** of the recipient.
If not rate-limited per key, enumerates account-holder names at scale. → **Low–Medium PII**.
(Bank-name-lookup is a deliberate feature; value depends on absent per-key limiting.)

## Precise test recipe (run when valid sandbox keys available)
1. On `sandbox.api.yellowcard.io/business` (NOT Cloudflare-blocked; no IP whitelist), obtain a
   sandbox API key + secret in the Treasury Portal (own test account).
2. Build a correct `YcHmacV1` signature per the documented message format (or use the official
   `@yellowcard/sdk` / SDK samples in the repo).
3. **H1:** as partner A, create your own send; note your `sequenceId`; then attempt
   `GET /business/payments/{sequenceId}` for (i) your own id [control → 200], (ii) a neighbouring
   / other-tenant / slightly-modified `sequenceId` [probe]. Diff status/body.
4. **H2:** capture a `submit-payment` request; replay it unchanged after `expiresAt`. Observe
   double-accept / duplicate creation.
5. **H3:** call `resolve-bank-account` repeatedly with distinct `accountNumber`s under one key;
   observe whether per-key rate limiting exists.
6. Record full request/response pairs; redact any real PII in evidence per program expectations;
   use ONLY numbers/identifiers you are entitled to create in the sandbox.

## Repro hygiene / scope compliance
- Sandbox only; no production; no real customer data; no other-tenant data unless it is an
  account/record you created.
- If H1 unexpectedly returns real third-party data, STOP immediately and report per the program's
  "halt and report if you obtain any authenticated/business account data" rule.
- Program: `bugbounty@yellowcard.io`.

## Blockers to completing validation
- **No valid API key / HMAC secret** — program scope is unauth-only, so obtaining one is
  disallowed from here; requires the researcher's own registered partner account.
- Sandbox is reachable but fully auth-gated (verified live: 401 on all `/business/*` paths).
