# SaaS Image Generator — Indonesian Payment Research

_Date: 2026-05-09_
_Status: Research complete_

---

## Key Finding: Stripe TIDAK Support QRIS

Stripe Indonesia hanya support **kartu kredit/debit + virtual account**. Tidak ada QRIS, GoPay, OVO, DANA, atau e-wallet Indonesia lainnya. Untuk market Indonesia, **wajib pakai gateway lokal**.

---

## Fee Comparison per Transaction

### Rp75,000 (~$5)

| Method | Gateway | Fee | Net Received | Margin |
|--------|---------|-----|-------------|--------|
| **QRIS** | Midtrans/Xendit | 0.7% = Rp525 | Rp74,475 | 99.3% |
| **GoPay** | Midtrans | 2% = Rp1,500 | Rp73,500 | 98% |
| **DANA** | Midtrans | 1.5% = Rp1,125 | Rp73,875 | 98.5% |
| **Virtual Account** | Midtrans | Rp4,000 flat | Rp71,000 | 94.7% |
| **Card (Intl)** | Stripe | ~$2.48 | ~Rp37,800 | 50.4% |
| **QRIS (cross-border)** | Tazapay | 1% = Rp750 | Rp74,250 | 99% |

### Rp150,000 (~$10)

| Method | Gateway | Fee | Net Received | Margin |
|--------|---------|-----|-------------|--------|
| **QRIS** | Midtrans/Xendit | 0.7% = Rp1,050 | Rp148,950 | 99.3% |
| **GoPay** | Midtrans | 2% = Rp3,000 | Rp147,000 | 98% |
| **DANA** | Midtrans | 1.5% = Rp2,250 | Rp147,750 | 98.5% |
| **Virtual Account** | Midtrans | Rp4,000 flat | Rp146,000 | 97.3% |
| **Card (Intl)** | Stripe | ~$4.95 | ~Rp105,000 | 70% |

**QRIS melalui gateway lokal menghemat 70-80% dibanding Stripe card processing.**

---

## Rekomendasi Final: Midtrans Starter (Individu) + Stripe (Intl)

### Phase 1 — MVP (sekarang)
- **Midtrans Starter Pack** — registrasi perorangan, cuma KTP
- QRIS (0.7%), GoPay (2%), Virtual Account (Rp4K)
- Tidak perlu PT, tidak perlu NPWP
- Webhook via Midtrans notification endpoint

### Phase 2 — Nanti
- Tambah NPWP pribadi → aktifkan kartu kredit/debit di Midtrans
- Atau tambah Stripe untuk international cards
- Migrasi ke PT kalau volume sudah tinggi (lebih banyak payment methods)

| Priority | Method | Gateway | Rationale |
|----------|--------|---------|-----------|
| 1 | QRIS | Midtrans | 0.7% fee, universal, best UX |
| 2 | GoPay | Midtrans | 2%, #1 e-wallet, native GoTo integration |
| 3 | Virtual Account | Midtrans | Rp4K flat, untuk user tanpa e-wallet |
| 4 | DANA | Midtrans | 1.5%, popular e-wallet |
| 5 | ShopeePay | Midtrans | 2%, large user base |
| 6 | Intl Cards | Stripe | 2.9%+$0.30, untuk customer internasional |

**Kenapa Midtrans bukan Xendit?**
- Midtrans punya **GoPay** (Xendit tidak list GoPay di pricing page)
- Lebih mudah onboarding (Starter Pack available)
- Documentation dalam Bahasa Indonesia
- Snap UI: hosted checkout page, mobile-optimized

---

## Architecture: Dual Gateway

```
[React Checkout Page]
        |
        v
[Payment Router — detect currency & location]
        |
   +----+----+
   |         |
   v         v
[Stripe]    [Midtrans Snap]
(intl)      (ID)
   |              |
   v              v
- Intl cards    - QRIS (0.7%)
- Apple Pay     - GoPay (2%)
- Google Pay    - OVO, DANA, ShopeePay
- SEPA/ACH      - Virtual Accounts (Rp4K)
                - Indomaret/Alfamart
```

---

## Registration Requirements

### Jika punya Indonesian entity (PT/PMA):
- **Xendit**: Passport + NPWP (individual + corporate) + business license + company deed
- **Midtrans**: Passport/KITAS + NPWP + PT documentation. Starter Pack (QRIS, GoPay, VA) lebih ringan requirements-nya

### Jika TIDAK punya Indonesian entity:
- **Tazapay**: No local entity needed, 1% QRIS fee, MAS-licensed (Singapore)
- **HitPay**: No local entity needed, cross-border QRIS, 1.5%+1% FX
- Trade-off: Higher fees, fewer payment methods, tapi zero setup overhead

---

## Compliance Notes

- **Bank Indonesia PBI 10/2025**: Domestic payment data must be processed through Indonesian infrastructure (NPG/GPN). Local gateways handle this automatically.
- **Data localization**: Local gateway = compliant. Cross-border provider = verify they have Indonesian data processing.
- **MDR QRIS**: 0% untuk UMI (annual revenue < Rp300 juta), 0.7% untuk business lain.
- **AML/KYC**: Required for merchant registration di semua gateway.

---

## Impact on Architecture Document

Perubahan yang perlu di-update di `architecture.md`:
1. Payment flow: tambah Midtrans Snap integration
2. Database: stripe_sessions → payment_sessions (support multiple gateways)
3. Environment variables: tambah Midtrans config (SERVER_KEY, CLIENT_KEY, SNAP_URL)
4. Frontend: Midtrans Snap JS SDK + payment method selection UI
5. Webhook: tambah Midtrans notification handler
6. Product pricing: IDR denominations (Rp75,000, Rp150,000) selain USD

---

## Sources

- Xendit Pricing: https://xendit.co/en-id/pricing/
- Midtrans Pricing: https://midtrans.com/pricing
- Midtrans QRIS Docs: https://docs.midtrans.com/docs/introduction-qris-payment
- DOKU Pricing: https://www.doku.com/harga
- Tazapay QRIS: https://tazapay.com/payment-methods/qris
- HitPay QRIS: https://hitpayapp.com/qris
- Bank Indonesia MDR: https://www.bi.go.id/id/publikasi/ruang-media/cerita-bi/Pages/mdr-qris.aspx