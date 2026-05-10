# BRD Modul M9 — Nomination Status & EPB Confirmation

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m9-nomination-status-payment.md`](../../implementation/replit-handoff/m9-nomination-status-payment.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.9](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola proses setelah nominasi disubmit hingga customer melakukan upload bukti pembayaran pertama kali (EPB Confirmation). Siklus verifikasi payment selanjutnya dikelola oleh menu EPB & Invoice (lihat M9b di bawah).

## Tujuan

- Menyediakan visibilitas status nominasi secara real-time kepada customer.
- Menampilkan detail EPB dan mengarahkan customer ke menu EPB & Invoice untuk melakukan pembayaran.
- Memfasilitasi revisi nominasi jika STS Platform meminta perubahan data.

## Cakupan (In Scope)

*Status Tracking*
- Customer dapat melihat status nominasi: **Draft** → **Submitted** → **Pending** (Menunggu proses di STS Platform) → **Approved** / **Need Revision** → **Waiting Payment Verification** (setelah upload proof di M9b).
- Customer menerima notifikasi saat status nominasi berubah.
- Customer dapat melakukan revisi data nominasi jika status = Need Revision, kemudian re-submit ke STS Platform.

*EPB Detail & Navigasi ke M9b (Jika Approved)*
- Customer melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta detail data: schedule, dock, dan nominal yang harus dibayarkan (view-only).
- Saat STS mengirim webhook APPROVED, sistem **otomatis membuat record EPB di menu EPB & Invoice (M9b) dengan status Unpaid** — tanpa aksi customer.
- Customer menekan tombol **"Bayar EPB"** untuk dinavigasi ke menu EPB & Invoice (M9b) guna melakukan upload bukti pembayaran.

> **Batasan Modul:** M9 tidak menyediakan form upload bukti pembayaran. Upload proof dilakukan sepenuhnya di menu EPB & Invoice (M9b). Proses approval, resource assignment, scheduling, generate EPB, dan verifikasi pembayaran seluruhnya dilakukan oleh STS Platform. Siklus Unpaid → Waiting Payment Verification → Payment Reject → Paid dikelola di menu EPB & Invoice (M9b).

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-NP-01 | Sistem harus menerima status update nominasi dari STS Platform (Pending, Approved, Need Revision) dan menampilkannya ke customer. Status Pending ditampilkan dengan label "Menunggu proses di STS Platform". | System |
| FR-NP-02 | Jika nominasi berstatus Approved, customer harus dapat melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta detail: schedule, dock, dan nominal yang harus dibayarkan (view-only) | Customer |
| FR-NP-03 | Customer harus dapat melihat detail jadwal (schedule, dock) setelah nominasi di-approve oleh STS Platform | Customer |
| FR-NP-04 | Sistem harus mengirim notifikasi ke customer saat status nominasi berubah | System |
| FR-NP-05 | Customer harus dapat melakukan revisi data nominasi jika status = Need Revision (branch False pada decision node "Is Approved?"), kemudian re-submit ke STS Platform | Customer |
| FR-NP-06 | Jika nominasi berstatus Approved, customer harus dapat melihat tombol "Bayar EPB" yang mengarah ke menu EPB & Invoice (M9b) untuk melakukan upload bukti pembayaran | Customer |
| FR-NP-07 | Saat STS Platform mengirim status Approved, sistem harus **otomatis membuat record EPB di menu EPB & Invoice (M9b) dengan status Unpaid**, tanpa memerlukan aksi dari customer | System |
| FR-NP-08 | Setelah customer upload bukti pembayaran di M9b, status nominasi di M9 harus ikut berubah menjadi "Waiting Payment Verification" secara bersamaan | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | — |
| VHF Operator | — |
| Security Operator | — |
| Admin Kepelabuhan | View |
| KSOP | — |
| Bea Cukai | — |
| Customer | Full |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M9](../BRD-LPS-V3-Analysis.md) & [Section 3.4.9](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/nomination-status-payment/`](../../module/nomination-status-payment/)
- Replit handoff: [`m9-nomination-status-payment.md`](../../implementation/replit-handoff/m9-nomination-status-payment.md)
- Modul terkait: [M8 — Nomination Submission](m8-nomination-submission.md), [M9b — EPB & Invoice](m9b-epb-invoice.md)
