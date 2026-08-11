<!-- heimdall-auto-checkpoint:begin -->
## Auto-checkpoint — 2026-08-11T05:33:47Z

> Written automatically at session end (mechanical, no LLM). Run `/hmd:save`
> for a richer human/LLM-authored handoff — it enriches this same file.

### Recent commits
- 284eef2 fix(policies): unify corporate office address to WeWork Galaxy 43, 560025
- a6226c0 heimdall: session-end checkpoint (3 files)
- 5c379ea fix(policies): Rally Promise refund calculated on Transaction Fees inclusive of GST
- 38f999c fix(policies): refine Rally Promise T&C
- ca73f32 heimdall: auto-checkpoint (5 files)
- 139ee99 feat(policies): add Rally Promise T&C page under Platform
- 44518e2 heimdall: session-end checkpoint (3 files)
- b0c7532 refund eta-2
- c7ab056 refund eta
- 2742e47 Merge pull request #1 from mewt-app/address-change

### Uncommitted changes (git status --porcelain)
```
 M .heimdall/.roster-cache.json
 M .heimdall/telemetry/events.ndjson
?? .heimdall/.roster-cache.json.20828.tmp
?? .heimdall/.roster-cache.json.22964.tmp
?? .heimdall/.roster-cache.json.5450.tmp
?? .heimdall/.roster-cache.json.67178.tmp
?? .heimdall/.roster-cache.json.86625.tmp
```

### Resume
Read this file, then `heimdall` to resume. If a goal is set above, restore it with `/goal <condition>`.
<!-- heimdall-auto-checkpoint:end -->

# Checkpoint — 2026-05-20

## TL;DR
Brought up beyound.in end-to-end: coming-soon → full landing with Wallet + Bills services, 18 policy pages scrubbed of Spaid branding, twilio SMS OTP wired across bills.beyound.in + wallet.beyound.in, history filter so legacy spaid/superpe wallet txns don't bleed into Beyound.

## Completed
- [x] Policy pages (18 HTML, terms/privacy/refund/grievance/about + wallet/bbps service variants + ipl/cc-leaderboard campaigns) (beyound: d26dbfb)
- [x] Landing redesign — hero, 2 service cards (wallet.beyound.in + bills.beyound.in), about, legal grid, footer (beyound: cb6325b)
- [x] Scroll indicator hidden globally (beyound: cb6325b)
- [x] webView=true → hide "All policies" back-link (beyound: 9fd0220)
- [x] Entity rename: Spaid E-Solutions → BeyoundExtra Tech Private Limited (beyound: d26dbfb, supermade: c397cf1)
- [x] Bills footer fix + scrub: links to beyound.in/policies/*, brand-aware ink color, /about-us+/policies/[slug] redirect (supermade: 07edbfc)
- [x] OTP send/validate via Mewt notification-otp wrapper, brand=beyound (mewt-backend-v3: PR #5496, #5497, #5499, #5501, #5503; tag v0.1.866)
- [x] Twilio direct-send (India-enabled creds inlined, services.py dep removed) — bypasses broken upstream notif-service twilio (mewt-backend-v3: a5fb0a28e on main)
- [x] Bills login OTP via twilio + 6-digit input + envelope error parsing (supermade: a0df954, ca72fa0, f41cf4f, 570e68a)
- [x] Wallet login OTP same path: send-otp route, NextAuth authorize on notification-otp/validate, OTP_LENGTH 6, sid threaded through LoginFlow (supermade: 6edb828)
- [x] Wallet history filter — drop pre-2026-05-13 IST txns for brand=beyound (supermade: 36ad4de)

## In Progress
- [ ] None — all session deliverables shipped.

## Not Started
- [ ] Beneficiary filter (if user wants the same cutoff applied to /payoutcc/beneficiaries)
- [ ] Bills history list (none exists today — only single order-status deep links)
- [ ] Notification-service twilio account India enablement (would let us drop the in-process bypass)

## Resume Instructions
1. If user reports any login flow regression, curl `POST https://api.spaid.in/backend/v2/authentication/notification-otp/send` with brand=beyound, provider=twilio, phone=<10 digits> — must return 200 `{sid, status:sent, brand:beyound, expires_in_seconds:300}`.
2. If twilio flips back to broken at upstream notif-service, edit `apps/{wallet,bills}/src/.../actions` to set `PROVIDER = "plivo"` (one-liner; SMS branding stays "Your beyound OTP is...").
3. If brand whitelist drops "beyound" again, re-add to `VALID_BRANDS` in `mewt-backend-v3:projects/mewt_backend/src/routers/notification_otp.py`.

## Key Context
- Three repos touched: `/Users/rj/Downloads/code/beyound` (static), `/Users/rj/Downloads/supermade` (Next.js apps), `/Users/rj/Downloads/code/mewt-backend-v3` (FastAPI).
- Backend prod deploys ONLY on tag `v*.*.*` push to main (`.github/workflows/build-deploy-gcp.yml`). develop → dev cluster only.
- Supermade prod deploys on push to main via `.github/workflows/build-and-deploy.yaml` (Build to GCP, ~5–6 min) + ArgoCD rollout (extra 2–4 min).
- bills.beyound.in / wallet.beyound.in shipped via per-brand Docker images, brand pinned at build time via `NEXT_PUBLIC_BRAND=beyound`.
- Twilio creds (India-enabled) live in `mewt-backend-v3` `projects/mewt_backend/src/routers/notification_otp.py` constants. Same account as the legacy ultrafast `appOrigin` path in `services.py`. (Don't write SIDs into docs/comments — GitHub push protection blocks.)
- Notification-otp router lives at `${BASE_URL}/authentication/notification-otp/{send,validate}` (BASE_URL = `https://api.spaid.in/backend/v2`).
- Same phone → same merchant_id across brands. That's why wallet history filter is by date, not by brand metadata.
- Plural wallet history returns formatted IST strings like `"20 May, 2026 09:15 AM"` — parse with the regex in `apps/wallet/src/app/api/wallet/transactions/route.ts`.
- DeleteAccountForm in bills still uses legacy 4-digit `/authentication/app-authenticate` — intentionally left untouched (separate flow).
- About Us + all policies are canonical on beyound.in; bills + wallet footer/nav link OUT to `https://beyound.in/policies/...`.

## Tech Stack & Patterns
- beyound.in repo: static HTML/CSS, GitHub Pages, CNAME `beyound.in`, Sora font, #059669 brand green.
- supermade apps: Next.js 15 App Router (wallet uses NextAuth credentials provider), Tailwind, brand-aware via CSS vars from `brandCssVars`.
- mewt-backend-v3: FastAPI, Redis-backed sessions, Pydantic schemas, asyncio.to_thread for blocking calls.
- Pattern: server actions return `{ok, ...} | {ok:false, error}` shapes — never throw, which Next.js surfaces as "Server Components render error" with redacted body.
- Pattern: notification-otp OTP cached under `notif_otp:{sid}` for 5min, validated by /validate which then mints session in same shape as legacy app-validate.

## Project Settings
- Parallelism: max 10, mostly mechanical edits — fine sequentially for this kind of work.
- Model routing: opus for code in this session (security-sensitive auth flow).
- Governance: hierarchical.
- Test command: `pnpm typecheck && pnpm test` (supermade) / none for beyound.in static / `pytest` for backend.
- Lint command: `pnpm lint` (supermade) — typecheck:ignore is on in next.config so type errors don't break build.
- Build command: `pnpm --filter @pay/wallet build` (per-brand via `NEXT_PUBLIC_BRAND` env).
- Deploy command: push to main → CI; for backend prod also `git tag v0.1.N+1 && git push origin v0.1.N+1`.
- Directories to avoid: `node_modules`, `.next`, `dist`, `.git`, `__pycache__`, `*.bak`.
- User preferences: caveman mode always on, code/commits/security in normal English, terse end-of-turn summaries, no emojis in code, no stubs/placeholders.
