# STATE — 2026-05-20

## Current phase
Idle. All deliverables for the Beyound brand launch (landing + policies + bills login + wallet login + history filter) shipped and verified live.

## Done (this session)
| Area | Commit / PR | Note |
|---|---|---|
| Policy pages (18 HTML) | beyound `f7ff7bc` → `d26dbfb` | Wallet + BBPS + generic + 2 campaigns; entity = BeyoundExtra Tech Private Limited |
| Landing redesign | beyound `cb6325b` | Hero + 2 service cards + about + legal + footer; scrollbar hidden; webView gate |
| Bills footer + scrub | supermade `07edbfc` | Beyound brand-aware ink color, links out to beyound.in/policies/* |
| Bills login OTP (twilio + 6-digit) | supermade `a0df954`, `570e68a`, `f41cf4f` | sid threaded, envelope error parsing |
| Wallet login OTP | supermade `6edb828` | NextAuth authorize uses notification-otp/validate, OTP_LENGTH 6 |
| Wallet history filter | supermade `36ad4de` | Per-brand cutoff drops pre-launch txns for beyound |
| Backend brand whitelist | mewt-backend-v3 PR #5496, #5497 | beyound added to VALID_BRANDS on develop + main |
| Backend twilio direct-send | mewt-backend-v3 PR #5499, #5501, #5503; tag `v0.1.866` | In-process send via India-enabled creds; bypasses broken upstream notif-service |

## In progress
None.

## Next (if asked)
- Apply same cutoff to beneficiary list (`/api/wallet/payoutcc/...`) if user wants beneficiaries scrubbed too.
- Get notification-service's twilio account enabled for India SMS, then we can drop `_send_otp_direct` and route everything upstream.
- Tag/release governance: backend prod ships only on `v*.*.*` push to main; build-deploy-gcp workflow validates tag is from main.

## Blockers
None.

## Decisions made
- About Us + policies live ONLY on beyound.in. bills/wallet apps link out (single source of truth for legal copy).
- Wallet brand pinned at build time via `NEXT_PUBLIC_BRAND=beyound`. Not runtime-detectable.
- Server actions return discriminated `{ok, ...}` shapes instead of throwing — Next.js wraps server-action throws as "Server Components render error" with redacted body in prod.
- Twilio India is routed in-process inside the notification-otp router because upstream notif-service's twilio account is not India-enabled (returns 400 "Invalid 'To' Phone Number" → 502 to client). Plivo still works upstream and is reserved for non-twilio providers.
- DeleteAccountForm in bills kept on legacy 4-digit `/authentication/app-authenticate` flow — separate from login, untouched.
- Wallet history filter is by date cutoff (per brand), not by brand metadata, because Plural's upstream txn payload doesn't carry brand info and same phone → same merchant_id across brands.

## Key files changed
**beyound (this repo):**
- `index.html` — full landing rewrite (was coming-soon)
- `policies/index.html`, `policies/{terms,privacy,refund-cancellation,grievance,about}.html`
- `policies/wallet/*`, `policies/bbps/*`, `policies/ipl-prediction.html`, `policies/cc-leaderboard.html`
- `policies/styles.css`

**supermade (separate repo):**
- `apps/bills/src/actions/index.tsx` — generateOtp/validateOtp via notification-otp wrapper
- `apps/bills/src/components/Modals/SignInModal.tsx` — sid state, 6-digit OTP input, envelope errors
- `apps/bills/src/components/{Footer,Header,LandingHero}.tsx` — links to beyound.in/policies
- `apps/bills/src/app/{about-us,policies/[slug]}/page.tsx` — redirects to beyound.in
- `apps/bills/src/brand/brands.ts`, `packages/brand-config/src/brands/beyound.ts` — entityName = BeyoundExtra Tech Private Limited
- `apps/wallet/src/app/api/auth/send-otp/route.ts` — notification-otp wrapper call
- `apps/wallet/src/auth.ts` — NextAuth authorize via notification-otp/validate
- `apps/wallet/src/components/login/{LoginFlow,OtpStep}.tsx` — sid, 6-digit
- `apps/wallet/src/app/api/wallet/transactions/route.ts` — per-brand history cutoff filter

**mewt-backend-v3 (separate repo):**
- `projects/mewt_backend/src/routers/notification_otp.py` — beyound whitelist + twilio direct-send + inline call (no services.py dep)
