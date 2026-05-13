# M9b — EPB (Customer Portal)

> **Catatan v3.3:** Modul ini sebelumnya bernama "EPB & Invoice" dengan scope gabungan. Sejak BRD v3.3 (Mei 2026), scope dipersempit menjadi **EPB only**. Bagian Invoice dipisah menjadi modul baru [M9c — Invoice](../invoice/). Folder ini tetap bernama `module/epb-invoice/` untuk menjaga konsistensi git history dan link existing.

## Overview
Modul dalam LPS Customer Portal untuk mengelola siklus pembayaran **EPB (Estimasi Perkiraan Biaya)** — tagihan awal yang dikeluarkan STS Platform setelah nominasi disetujui. Customer dapat membayar penuh atau parsial (minimum 1 USD ekuivalen IDR); shortfall otomatis menjadi Invoice di M9c.

Row `epb_payments` dibuat otomatis dengan status `Belum Dibayar` (UNPAID) saat STS mengirim webhook `APPROVED` ke M9. Customer kemudian membuka tab "EPB" di menu "EPB & Invoice" untuk input nominal pembayaran + upload payment proof. Status berubah jadi `Menunggu Verifikasi` (WAITING_PAYMENT_VERIFICATION). Siklus berlanjut hingga `Lunas` (PAID) atau loop via `Pembayaran Ditolak` (PAYMENT_REJECT).

## Boundaries

**In scope:**
- Tab "EPB" di menu "EPB & Invoice" — list seluruh EPB customer dengan status pembayaran terkini.
- Detail EPB (view-only, data dari STS Platform).
- **Partial payment**: customer dapat input nominal pembayaran (min 1 USD) saat klik "Bayar".
- Upload payment proof, baik untuk first payment (Belum Dibayar) maupun re-upload (Pembayaran Ditolak).
- Menerima webhook STS: `EPB_PAYMENT_REJECT`, `EPB_PAID`, `EPB_SHORTFALL_DETECTED`.
- Forward bukti bayar + paid_amount ke STS Platform via API untuk verifikasi.
- Update `nominations.status` paralel saat status `epb_payments` berubah.

**Out of scope:**
- Pembuatan row `epb_payments` awal (oleh M9 webhook handler saat STS event APPROVED).
- Invoice (settlement) — dikelola oleh [M9c — Invoice](../invoice/).
- Billing calculations, invoice generation, payment reconciliation (STS Platform).
- Nomination approval logic (STS Platform).
- Dashboard summary (M10).

## Dependencies

| Dependency | Reason |
|-----------|--------|
| M9 Nomination Status & EPB Confirmation | Creates initial `epb_payments` row (status `UNPAID`) saat STS webhook APPROVED diterima |
| M7 Customer Authentication | Customer must be authenticated |
| STS Platform Webhook (inbound) | Receives EPB status updates: `EPB_PAYMENT_REJECT`, `EPB_PAID`, `EPB_SHORTFALL_DETECTED` |
| STS Platform API (outbound) | Sends uploaded payment proofs + paid_amount for verification |

## Downstream

| Module | Dependency |
|--------|-----------|
| M9c Invoice | Menerima Invoice baru saat M9b webhook `EPB_SHORTFALL_DETECTED` diteruskan |
| M10 Customer Dashboard & Monitoring | Displays EPB payment summary on dashboard |

## Key Decisions

- M9b adalah modul terpisah dari M9. M9 menampilkan EPB dan mengarahkan customer ke M9b; M9b mengelola seluruh siklus payment.
- Row `epb_payments` dibuat oleh M9 webhook handler saat event `APPROVED` diterima dari STS, dengan status `Belum Dibayar` (UNPAID). Customer belum upload apa-apa saat ini.
- **Partial payment diperbolehkan:** customer input nominal pembayaran (min 1 USD ekuivalen IDR) saat klik "Bayar". STS Platform akan menentukan Lunas/Ditolak; saat Lunas dengan paid_amount < total_amount → STS push webhook `EPB_SHORTFALL_DETECTED` → M9c create Invoice row.
- Upload proof (pertama kali maupun re-upload) **seluruhnya dilakukan di M9b** melalui satu endpoint yang sama, dibedakan oleh current status (`Belum Dibayar` atau `Pembayaran Ditolak`).
- Setelah customer upload proof: `epb_payments.status` → `Menunggu Verifikasi` DAN `nominations.status` → `WAITING_PAYMENT_VERIFICATION` (paralel, satu transaksi DB).
- Saat EPB Lunas, `nominations.status` → `PAYMENT_CONFIRMED` (parsial OK — Invoice tidak block nominasi).
- LPS does **not** verify payments. Semua keputusan verifikasi dari STS Platform via webhook.
- STS webhook events: `EPB_PAYMENT_REJECT`, `EPB_PAID`, `EPB_SHORTFALL_DETECTED`. Status `Menunggu Verifikasi` di-set langsung oleh LPS saat customer upload — tidak perlu webhook konfirmasi.
- **Label status berbahasa Indonesia natural (v3.3):** Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas (menggantikan istilah teknis sebelumnya).

---

## UI/UX Design

**Surface A — Customer Portal** (Bahasa Indonesia).

UI berada di tab "EPB" dalam menu "EPB & Invoice" di sidebar Customer Portal. Menu sidebar tetap satu (tidak split jadi dua item); pemisahan EPB vs Invoice ada di sub-tab dalam halaman.

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m9b-epb-invoice-ui.md`](../../implementation/design/m9b-epb-invoice-ui.md)

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
