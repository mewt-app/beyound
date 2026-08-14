---
name: project-beyound-checkout
description: Beyound Checkout is in developer preview; its facts come from the rallycheckout reference repo, whose internal vocabulary must never reach public pages
metadata:
  type: project
---

Beyound Checkout (added 2026-08-14) is a merchant checkout product on beyound.in:
shoppers select Beyound at an ecommerce brand's checkout, approve a card once via
a bank-authenticated e-mandate, then repeat payments up to Rs 25,000 are one tap.
It is in **developer preview** — not generally available, no public API base URL,
credentials issued on request via support@beyound.in.

Product facts live in `/Users/rj/Downloads/code/rallycheckout` (a Vite/React
reference implementation + `docs/analysis/api-contract.md`), NOT in this repo.

**Why:** that repo is internal and uses vocabulary that is not the public brand —
"Rally", "SuperCash", and "bolting"/"bolted" for the e-mandate. It also contains
no marketing copy, no public docs URL, and no claims about conversion lift, EMI,
settlement times, fees, or ecommerce platform plugins.

**How to apply:** when editing `/checkout/` pages, source every endpoint, field
and error code from `api-contract.md` and grep to confirm. Translate internal
terms (bolted card → "approved card" / "one tap"). Do not write regulatory
compliance claims ("RBI-compliant") for this product while it is unlaunched —
"bank-authenticated" is the approved wording. Any stat, percentage, plugin or
timeline claim is an invention and must be rejected.
