# Project Tracker

_Current mode: project_

## Mode Switching

- **Enter Project Mode**: Say "project mode", "mulai project", or assign a project task
- **Exit to Normal Mode**: Say "normal mode", "stop project", or project is completed

## Projects

### [1] SaaS Image Generator (kie.ai)
- **Status**: ACTIVE
- **Started**: 2026-05-08
- **Last activity**: 2026-05-09 — QA complete, 7 bugs found and all fixed
- **Phase**: Implementation (M1-M5 done, QA bugs fixed)
- **Next**: Deployment to VPS
- **Payment**: Midtrans Starter Pack (QRIS 0.7%, GoPay 2%, VA Rp4K) — registrasi perorangan KTP saja, tanpa PT. Stripe nanti untuk intl cards.
- **Registration**: Cukup KTP, tanpa NPWP, tanpa PT. Rekening bank pribadi.
- **Description**: SaaS image editor using kie.ai nano-banana-edit model. One-time purchase credits.
- **Pricing**: Rp75.000 = 400 credits (100 gens) | Rp150.000 = 1000 credits (250 gens) | 5 free gens for new users
- **Model**: nano-banana-edit (4 credits/gen, $0.02/gen cost)
- **Decisions**: One-time purchase (no expiry), 5 free gens, FastAPI + React
- **Files**: `projects/saas-image-gen/credit-pricing-analysis.md`
- **QA Bugs Fixed**:
  1. KieClient crash on null response → proper error handling
  2. Rate limiting not wired → added to auth + generation routes
  3. Rate limit ID prefix → fixed "rl_" prefix
  4. Generation 307 redirect → removed trailing slash
  5. total_paid_credits → renamed to total_credits_granted
  6. Security headers → added X-Frame-Options, X-Content-Type-Options, etc.
  7. No automated tests (noted, not blocking)
- **Blockers**: None

---

## Instructions for Claude (Project Mode)

When entering project mode:
1. Update the mode line above to `project`
2. Create a project entry below with status, phase, and next steps
3. After every significant action, update the project entry

When a project is completed:
1. Change status to COMPLETED
2. Move to a "Completed Projects" section at the bottom
3. If no other active projects, switch back to normal mode

## Completed Projects

_(None yet)_