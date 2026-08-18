# Subdomain takeover candidate — track.yellowcard.io / track.go.yellowcard.io (Snov.io)

## Summary
Both `track.yellowcard.io` and `track.go.yellowcard.io` are CNAME records pointing at
`sn ov-stream.com` — the Snov.io email-tracking stream service — and both serve the
Snov.io default "Custom tracking domain" placeholder page ("Congratulations, Snovian!
Your custom tracking domain is working now!").

## Evidence
1. DNS:
   - `track.yellowcard.io is an alias for snov-stream.com.` (52.2.249.45, 54.89.35.162)
   - `track.go.yellowcard.io is an alias for snov-stream.com.` (same addresses)
2. HTTP GET / on both hosts -> 200, 303B, identical body:
   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <title>Custom tracking domain</title>
   </head>
   <body style="text-align: center;">
           <h2>Congratulations, Snovian!</h2>
           <p style="font-size: 16px">Your custom tracking domain is working now!</p>
   </body>
   </html>
   ```

## Root cause / attack
Snov.io custom tracking domains are claimed per-account. A domain pointing at
snov-stream.com that is not actively bound to an account serves this placeholder and
can be claimed by any Snov.io user. Claiming it would let an attacker:
- hijack/replace tracking links in Yellow Card's outbound email (email opens/click
  telemetry, link redirects),
- craft official-looking `track.yellowcard.io/<uuid>` links for phishing,
- impersonate the brand on a subdomain of a financial brand's apex domain.

## Impact
Medium (per SendGrid/Postmark custom-tracking-domain takeover precedent): email link
hijack + phishing on an official subdomain of a licensed crypto exchange.

## Status
PROOF INCOMPLETE: the claim step requires registering a Snov.io account and adding the
domain — HUMAN step (also confirms claimability vs dormant-but-bound).
SCOPE NOTE: hosts not in scope.yml targets; verify against the program's published
asset list (bugbounty@yellowcard.io) before submitting.
