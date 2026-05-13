# M9c — Invoice (Customer Portal)

> **Modul baru di v3.3:** Sebelumnya bagian Invoice tergabung dengan M9b (EPB & Invoice). Sejak v3.3 dipisah menjadi modul standalone. M9c menangani **Invoice** (tagihan settlement), M9b menangani **EPB** (tagihan awal). UI keduanya tetap dalam satu menu "EPB & Invoice" di sidebar Customer Portal dengan tab internal terpisah.

## Overview
Modul dalam LPS Customer Portal untuk mengelola pembayaran **Invoice** — tagihan settlement yang muncul dari (a) shortfall pembayaran EPB sebelumnya atau (b) Additional Service yang dipakai customer selama voyage. Invoice di-generate otomatis oleh STS Platform via webhook dan dikelola di LPS sebagai interface customer.

Tidak seperti EPB yang dibuat saat nominasi approved, Invoice baru muncul **setelah** event tertentu:
- **EPB_SHORTFALL_DETECTED**: customer membayar EPB parsial dan STS mark EPB Lunas → shortfall otomatis jadi Invoice.
- **ADDITIONAL_SERVICE_INVOICE**: customer memakai layanan tambahan selama voyage (Tank Cleaning, Bunkering, dll) → STS push Invoice baru.

## Boundaries

**In scope:**
- Tab "Invoice" di menu "EPB & Invoice" — list seluruh Invoice customer.
- Detail Invoice (view-only) dengan info source (Shortfall EPB / Additional Service), parent reference, amount, due date.
- Upload payment proof, baik untuk first payment (Belum Dibayar) maupun re-upload (Pembayaran Ditolak).
- Menerima webhook STS: `EPB_SHORTFALL_DETECTED`, `ADDITIONAL_SERVICE_INVOICE`, `INVOICE_PAYMENT_REJECT`, `INVOICE_PAID`.
- Forward bukti bayar ke STS Platform via API untuk verifikasi.

**Out of scope:**
- Generate Invoice secara mandiri (LPS tidak generate — semua dari webhook STS).
- EPB lifecycle — dikelola oleh [M9b — EPB](../epb-invoice/).
- Memblokir lifecycle nominasi (Invoice = settlement separate; tidak mengubah `nominations.status`).
- Billing calculations, payment reconciliation (STS Platform).
- Dashboard summary (M10).

## Dependencies

| Dependency | Reason |
|-----------|--------|
| M9b EPB | Sumber webhook `EPB_SHORTFALL_DETECTED` (handler M9b memforward ke M9c create handler) |
| M8 Nomination Submission | Customer pilih Additional Service di nominasi; STS gunakan info ini untuk generate `ADDITIONAL_SERVICE_INVOICE` |
| M7 Customer Authentication | Customer must be authenticated |
| STS Platform Webhook (inbound) | Receives Invoice creation events + status updates |
| STS Platform API (outbound) | Sends uploaded payment proofs for verification |

## Downstream

| Module | Dependency |
|--------|-----------|
| M10 Customer Dashboard & Monitoring | Displays Invoice payment summary on dashboard (jumlah Invoice Perlu Tindakan) |
| M9b EPB | Cross-link: detail EPB yang parsial menampilkan link ke Invoice anaknya |

## Key Decisions

- **Invoice tidak memblokir lifecycle nominasi.** Status `nominations.status` = `PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (M9b), bukan oleh Invoice. Invoice adalah settlement separate yang dapat berjalan paralel/setelah voyage selesai.
- Semua Invoice **berasal dari webhook STS** — LPS tidak generate Invoice secara mandiri.
- Source Invoice dibedakan dengan kolom `source` enum: `EPB_SHORTFALL` atau `ADDITIONAL_SERVICE`. Setiap source punya field tambahan:
  - `EPB_SHORTFALL`: `parent_epb_id` (FK ke `epb_payments`).
  - `ADDITIONAL_SERVICE`: `service_keys[]` (array dari nominations_additional_services keys).
- Upload proof (pertama kali maupun re-upload) **seluruhnya dilakukan via satu endpoint yang sama**, dibedakan oleh current status (`Belum Dibayar` atau `Pembayaran Ditolak`).
- LPS does **not** verify payments. Semua keputusan verifikasi dari STS Platform via webhook.
- STS webhook events: `EPB_SHORTFALL_DETECTED`, `ADDITIONAL_SERVICE_INVOICE`, `INVOICE_PAYMENT_REJECT`, `INVOICE_PAID`. Status `Menunggu Verifikasi` di-set langsung oleh LPS saat customer upload — tidak perlu webhook konfirmasi.
- **Tidak ada partial payment di Invoice** (berbeda dengan EPB). Invoice harus dibayar penuh — ini sudah merupakan settlement.
- **Label status berbahasa Indonesia natural:** Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas (sama dengan EPB untuk konsistensi UX).

---

## UI/UX Design

**Surface A — Customer Portal** (Bahasa Indonesia).

UI berada di tab "Invoice" dalam menu "EPB & Invoice" di sidebar Customer Portal. Menu sidebar tetap satu (tidak split jadi dua item); pemisahan EPB vs Invoice ada di sub-tab dalam halaman.

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m9c-invoice-ui.md`](../../implementation/design/m9c-invoice-ui.md)

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
