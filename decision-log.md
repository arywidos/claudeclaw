# Decision Log

Records of key decisions made during projects. When resuming a project, read this file to understand **why** choices were made — not just what was done.

---

## [1] SaaS Image Generator (kie.ai)

### D-001: One-time purchase credits (no expiry)
- **Date**: 2026-05-08
- **Context**: Choosing monetization model
- **Options considered**:
  1. Monthly subscription with monthly credits
  2. One-time purchase credits with expiry
  3. One-time purchase credits, no expiry
- **Chosen**: Option 3 — one-time purchase, no expiry
- **Why**: Lower friction for Indonesian market. Users distrust subscriptions. Credits that expire feel like a scam. Simple = better conversion.
- **Impact**: No recurring revenue, but higher initial conversion. Need volume.

### D-002: FastAPI + React stack
- **Date**: 2026-05-08
- **Context**: Choosing tech stack
- **Options considered**:
  1. Django + React
  2. FastAPI + React
  3. Next.js (full stack)
- **Chosen**: Option 2 — FastAPI + React
- **Why**: Async-first (needed for polling kie.ai generation status), lightweight, fast to iterate. Team stronger in Python backend. React for SPA fits the editor UX.

### D-003: Midtrans for payments (not Stripe)
- **Date**: 2026-05-08
- **Context**: Payment processor for Indonesian market
- **Options considered**:
  1. Stripe only
  2. Midtrans only
  3. Midtrans + Stripe (Midtrans for ID, Stripe for intl)
- **Chosen**: Option 3 — Midtrans for Indonesian payments, Stripe later for international cards
- **Why**: Midtrans supports QRIS, GoPay, VA — essential for ID market. Stripe doesn't support these. Midtrans Starter Pack only needs KTP, no PT required. Stripe for international users added later.

### D-004: 5 free generations for new users
- **Date**: 2026-05-08
- **Context**: Deciding freemium model
- **Options considered**:
  1. No free tier
  2. 3 free gens
  3. 5 free gens
  4. 10 free gens
- **Chosen**: Option 3 — 5 free generations
- **Why**: Enough to let users experience the product and see quality. Not so many that abuse is attractive. Cost per gen is ~$0.02, so 5 gens = $0.10 per user — acceptable acquisition cost.

### D-005: Credit-based pricing (not per-image)
- **Date**: 2026-05-08
- **Context**: Pricing unit model
- **Chosen**: 4 credits per generation (1 credit = Rp187.50 at starter pack)
- **Why**: Credits give flexibility to change model pricing later without changing displayed prices. Also allows future features to cost different amounts (e.g., HD = 8 credits).

---

_(Add new decisions below using the format above)_