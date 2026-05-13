# M9b — EPB & Invoice (Customer Portal)

## Overview
A dedicated menu within the LPS Customer Portal untuk mengelola seluruh siklus pembayaran EPB. Row `epb_payments` dibuat otomatis dengan status `UNPAID` saat STS mengirim webhook `APPROVED` ke M9. Customer kemudian membuka M9b untuk melakukan upload payment proof, yang mengubah status menjadi `WAITING_PAYMENT_VERIFICATION`. Siklus berlanjut hingga `PAID` atau melalui loop `PAYMENT_REJECT`. Semua keputusan verifikasi dibuat oleh STS Platform via webhook.

## Boundaries
**In scope:**
- Displaying the list of all EPBs belonging to a customer with their current payment status (Unpaid, Waiting Payment Verification, Payment Reject, Paid)
- Viewing EPB & Invoice detail (view-only, data sourced from STS Platform via M9)
- Unpaid flow: customer klik "Pay" → upload payment proof → submit → status berubah ke `WAITING_PAYMENT_VERIFICATION`
- Waiting Payment Verification flow: view-only, menunggu keputusan STS Platform
- Payment Reject flow: customer klik "Revision Data" → upload ulang proof → re-submit
- Paid flow: view-only confirmation bahwa pembayaran selesai
- Receiving payment status update webhooks dari STS Platform (`EPB_PAYMENT_REJECT`, `EPB_PAID`)
- Forwarding uploaded payment proofs ke STS Platform via API
- Update `nominations.status` secara paralel saat status `epb_payments` berubah

**Out of scope:**
- Pembuatan row `epb_payments` awal (dilakukan oleh M9 webhook handler saat APPROVED — buat row dengan status `UNPAID`)
- Billing calculations, invoice generation, or payment reconciliation (STS Platform)
- Nomination approval logic (STS Platform)
- Dashboard summary (M10)

## Dependencies
| Dependency | Reason |
|-----------|--------|
| M9 Nomination Status & EPB Confirmation | Creates the initial `epb_payments` row (status `UNPAID`) saat STS webhook APPROVED diterima |
| M7 Customer Authentication | Customer must be authenticated |
| STS Platform Webhook (inbound) | Receives payment status updates: EPB_PAYMENT_REJECT, EPB_PAID |
| STS Platform API (outbound) | Sends uploaded payment proofs for verification |

## Downstream
| Module | Dependency |
|--------|-----------|
| M10 Customer Dashboard & Monitoring | Displays EPB payment summary on dashboard |

## Key Decisions
- M9b adalah modul terpisah dari M9. M9 menampilkan EPB dan mengarahkan customer ke M9b; M9b mengelola seluruh siklus payment.
- Row `epb_payments` dibuat oleh M9 webhook handler saat event `APPROVED` diterima dari STS, dengan status `UNPAID`. Customer belum upload apa-apa saat ini.
- Upload proof (pertama kali maupun re-upload) **seluruhnya dilakukan di M9b** melalui satu endpoint yang sama, dibedakan oleh current status (`UNPAID` atau `PAYMENT_REJECT`).
- Setelah customer upload proof: `epb_payments.status` → `WAITING_PAYMENT_VERIFICATION` DAN `nominations.status` → `WAITING_PAYMENT_VERIFICATION` (diupdate oleh M9b backend secara bersamaan).
- LPS does **not** verify payments. All verification decisions come from STS Platform via webhook.
- STS Platform webhook events handled here: `EPB_PAYMENT_REJECT`, `EPB_PAID`. (`EPB_PENDING_REVIEW` tidak dibutuhkan — M9b sendiri yang set `WAITING_PAYMENT_VERIFICATION` saat proof diupload.)

---

## UI/UX Design

**Surface A — Customer Portal** (Bahasa Indonesia).

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m9b-epb-invoice-ui.md`](../../implementation/design/m9b-epb-invoice-ui.md)

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
