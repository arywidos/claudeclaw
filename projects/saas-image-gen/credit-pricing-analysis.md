# SaaS Image Generator — Credit Pricing Analysis

_Date: 2026-05-09_

## Biaya Dasar

| Item | Value |
|---|---|
| kie.ai cost per 1000 credits | $5.00 |
| nano-banana-edit per gen | 4 credits |
| Cost per generation | $0.02 |
| Stripe fee | 2.9% + $0.30 per transaction |

---

## Tier $5 — Scenario Analysis

| Credits | Generations | API Cost | Stripe Fee | Total Cost | Profit | Margin |
|---------|-------------|----------|------------|-------------|--------|--------|
| 200 | 50 | $1.00 | $0.445 | $1.445 | $3.56 | 71% |
| 400 | 100 | $2.00 | $0.445 | $2.445 | $2.56 | 51% |
| 600 | 150 | $3.00 | $0.445 | $3.445 | $1.56 | 31% |
| 800 | 200 | $4.00 | $0.445 | $4.445 | $0.56 | 11% |

## Tier $10 — Scenario Analysis

| Credits | Generations | API Cost | Stripe Fee | Total Cost | Profit | Margin |
|---------|-------------|----------|------------|-------------|--------|--------|
| 500 | 125 | $2.50 | $0.59 | $3.09 | $6.91 | 69% |
| 800 | 200 | $4.00 | $0.59 | $4.59 | $5.41 | 54% |
| 1000 | 250 | $5.00 | $0.59 | $5.59 | $4.41 | 44% |
| 1200 | 300 | $6.00 | $0.59 | $6.59 | $3.41 | 34% |
| 1600 | 400 | $8.00 | $0.59 | $8.59 | $1.41 | 14% |

---

## Rekomendasi: Best Profit + Good Value

| Tier | Credits | Gens | Profit/User | Margin | Value Ratio |
|------|---------|------|-------------|--------|-------------|
| **$5** | **400** | **100** | **$2.56** | **51%** | 20 gens/$ |
| **$10** | **1000** | **250** | **$4.41** | **44%** | 25 gens/$ |

**Kenapa kombinasi ini?**
- $10 memberi 25% lebih banyak gens per dollar — incentive untuk upgrade
- Margin sehat di kedua tier (51% dan 44%)
- 100 gens/month di $5 cukup untuk casual user (3-4 edits/hari)
- 250 gens/month di $10 cukup untuk power user (8 edits/hari)

---

## Underutilization Profit (Subscription Model)

Kalau langganan bulanan dengan credit reset, rata-rata user hanya pakai 50-70% credits:

| Scenario | $5 Actual Profit | $10 Actual Profit |
|---|---|---|
| 100% used | $2.56 | $4.41 |
| 70% used | $2.91 | $5.23 |
| 50% used | $3.13 | $5.83 |

---

## Key Decisions (Pending User Input)

- [ ] Confirm pricing: $5/400 credits, $10/1000 credits
- [ ] Subscription vs one-time purchase
- [ ] Credit rollover policy (reset monthly or accumulate?)
- [ ] Free tier? (e.g., 5-10 free generations for trial)
- [ ] Tech stack choice (Vercel/Cloudflare/Hetzner)
- [ ] Payment processor (Stripe or alternative for lower micro-transaction fees)