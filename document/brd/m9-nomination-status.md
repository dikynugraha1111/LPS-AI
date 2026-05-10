# BRD Modul M9 — Nomination Status & EPB Confirmation

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m9-nomination-status-payment.md`](../../implementation/replit-handoff/m9-nomination-status-payment.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.9](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola proses setelah nominasi disubmit hingga customer dinavigasi ke menu EPB & Invoice untuk melakukan pembayaran. Siklus verifikasi payment selanjutnya dikelola oleh M9b.

## Tujuan

- Menyediakan visibilitas status nominasi secara real-time kepada customer.
- Menampilkan detail EPB dan mengarahkan customer ke menu EPB & Invoice (M9b) untuk melakukan pembayaran.
- Memfasilitasi revisi nominasi jika STS Platform meminta perubahan data.

## Cakupan (In Scope)

### Status Tracking

Customer dapat melihat status nominasi dengan lifecycle berikut:

```
Draft → Submitted → Pending → Approved / Need Revision → Waiting Payment Verification
```

| Status | Label Tampilan | Keterangan |
|--------|---------------|------------|
| Draft | Draft | Belum disubmit |
| Submitted | Submitted | Sudah disubmit ke STS |
| Pending | Menunggu proses di STS Platform | Diproses STS |
| Approved | Approved | STS menyetujui nominasi |
| Need Revision | Need Revision | STS minta revisi data |
| Waiting Payment Verification | Waiting Payment Verification | Setelah upload proof di M9b |

### EPB Detail & Navigasi ke M9b (Jika Approved)

- Customer melihat EPB (Estimasi Perkiraan Biaya) yang digenerate STS Platform: schedule, dock, nominal (view-only).
- Saat STS mengirim webhook APPROVED, sistem **otomatis membuat record EPB di M9b dengan status Unpaid** — tanpa aksi customer.
- Customer menekan tombol **"Bayar EPB"** → dinavigasi ke M9b.

## Batasan Modul

- M9 **tidak** menyediakan form upload bukti pembayaran.
- Upload proof dilakukan sepenuhnya di M9b.
- Proses approval, scheduling, generate EPB, dan verifikasi pembayaran seluruhnya dilakukan STS Platform.

## STS Webhook yang Ditangani M9

| Event | Aksi LPS |
|-------|----------|
| APPROVED | Update status nominasi + otomatis buat record epb_payments (UNPAID) di M9b |
| NEED_REVISION | Update status nominasi ke Need Revision |

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-NP-01 | Sistem harus menerima status update nominasi dari STS Platform (Pending, Approved, Need Revision) dan menampilkannya ke customer. Status Pending ditampilkan dengan label "Menunggu proses di STS Platform". | System |
| FR-NP-02 | Jika nominasi berstatus Approved, customer harus dapat melihat EPB yang digenerate STS Platform beserta detail: schedule, dock, dan nominal yang harus dibayarkan (view-only) | Customer |
| FR-NP-03 | Customer harus dapat melihat detail jadwal (schedule, dock) setelah nominasi di-approve oleh STS Platform | Customer |
| FR-NP-04 | Sistem harus mengirim notifikasi ke customer saat status nominasi berubah | System |
| FR-NP-05 | Customer harus dapat melakukan revisi data nominasi jika status = Need Revision, kemudian re-submit ke STS Platform | Customer |
| FR-NP-06 | Jika nominasi berstatus Approved, customer harus dapat melihat tombol "Bayar EPB" yang mengarah ke menu EPB & Invoice (M9b) | Customer |
| FR-NP-07 | Saat STS Platform mengirim status Approved, sistem harus **otomatis membuat record EPB di M9b dengan status Unpaid**, tanpa memerlukan aksi dari customer | System |
| FR-NP-08 | Setelah customer upload bukti pembayaran di M9b, status nominasi di M9 harus ikut berubah menjadi "Waiting Payment Verification" secara bersamaan | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| Admin Kepelabuhan | View |
| Customer | Full |
| SuperAdmin | View |
| Role lainnya | — |

---

## Referensi

- BRD Utama: [Section 2.1 M9](../BRD-LPS-V3-Analysis.md) & [Section 3.4.9](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/nomination-status-payment/`](../../module/nomination-status-payment/)
- Replit handoff: [`m9-nomination-status-payment.md`](../../implementation/replit-handoff/m9-nomination-status-payment.md)
- Modul terkait: [M8 — Nomination Submission](m8-nomination-submission.md), [M9b — EPB & Invoice](m9b-epb-invoice.md)
