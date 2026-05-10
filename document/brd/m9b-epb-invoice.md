# BRD Modul M9b — EPB & Invoice (Customer Portal)

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m9b-epb-invoice.md`](../../implementation/replit-handoff/m9b-epb-invoice.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.9b](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Menu dalam Customer Portal LPS yang berfungsi sebagai interface pemantauan status pembayaran EPB dan manajemen revisi bukti pembayaran. Data EPB di-generate oleh STS Platform; LPS hanya menampilkan dan mengelola proses upload/revisi bukti bayar.

## Tujuan

- Menyediakan visibilitas status payment EPB bagi customer.
- Memfasilitasi upload ulang bukti pembayaran jika sebelumnya ditolak.
- Menjadi single place bagi customer untuk memantau semua EPB & Invoice mereka.

## Cakupan (In Scope)

### 4 Status Payment

| Status | Deskripsi | Aksi Customer |
|--------|-----------|---------------|
| **Unpaid** | Record EPB dibuat otomatis saat nominasi di-approve STS. | Click "Pay" → Upload Proof → Submit |
| **Waiting Payment Verification** | Bukti sudah disubmit, menunggu verifikasi STS Platform. | View-only |
| **Payment Reject** | Bukti ditolak STS Platform. | Click "Revision Data" → Upload ulang → Submit |
| **Paid** | Pembayaran dikonfirmasi STS Platform. | View-only |

### Flow Per Status

```
Unpaid
  └─ Click Pay → Upload Payment Proof → Submit
       └─ Waiting Payment Verification (view-only)
            ├─ Payment Reject → Click Revision Data → Upload ulang → Submit → Waiting Payment Verification
            └─ Paid (view-only, selesai)
```

### STS Webhook yang Ditangani M9b

| Event | Aksi LPS |
|-------|----------|
| EPB_PAYMENT_REJECT | Update epb_payments.status → PAYMENT_REJECT |
| EPB_PAID | Update epb_payments.status → PAID |

> Status WAITING_PAYMENT_VERIFICATION di-set langsung oleh LPS saat customer upload proof — tidak perlu webhook dari STS.

## Batasan Modul

- Verifikasi pembayaran (keputusan Reject/Paid) sepenuhnya dilakukan oleh STS Platform.
- LPS hanya menerima status update dari STS Platform dan menampilkannya ke customer.
- Satu endpoint upload melayani dua flow (Pay dari UNPAID dan Revision Data dari PAYMENT_REJECT) — dibedakan oleh status saat ini.
- Setelah customer upload di M9b, status nominasi di M9 juga ikut diupdate dalam satu transaksi DB.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-EI-01 | Sistem harus menyediakan menu EPB & Invoice di Customer Portal yang menampilkan daftar seluruh EPB customer beserta status pembayaran terkini (Unpaid, Waiting Payment Verification, Payment Reject, Paid) | Customer |
| FR-EI-02 | Customer harus dapat melihat detail EPB dan Invoice (view-only) dengan mengklik item di daftar EPB & Invoice | Customer |
| FR-EI-03 | Untuk EPB berstatus **Unpaid**, customer harus dapat memulai proses pembayaran dengan mengklik tombol "Pay", kemudian mengupload bukti pembayaran dan mengisi data terkait, kemudian Submit | Customer |
| FR-EI-04 | Untuk EPB berstatus **Waiting Payment Verification**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer |
| FR-EI-05 | Untuk EPB berstatus **Payment Reject**, customer harus dapat mengklik "Revision Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer |
| FR-EI-06 | Untuk EPB berstatus **Paid**, sistem harus menampilkan detail dalam mode view-only sebagai konfirmasi bahwa pembayaran telah selesai | Customer |
| FR-EI-07 | Sistem harus menerima status update pembayaran dari STS Platform (Payment Reject / Paid) dan memperbarui tampilan di menu EPB & Invoice secara real-time. STS hanya mengirim dua event: EPB_PAYMENT_REJECT dan EPB_PAID. | System |
| FR-EI-08 | Sistem harus mengirimkan data bukti pembayaran ke STS Platform via API untuk proses verifikasi | System |
| FR-EI-09 | Setelah customer berhasil upload bukti pembayaran, sistem harus mengupdate status nominasi terkait di M9 menjadi "Waiting Payment Verification" secara bersamaan dalam satu transaksi | System |

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

- BRD Utama: [Section 2.1 M9b](../BRD-LPS-V3-Analysis.md) & [Section 3.4.9b](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/epb-invoice/`](../../module/epb-invoice/)
- Replit handoff: [`m9b-epb-invoice.md`](../../implementation/replit-handoff/m9b-epb-invoice.md)
- Modul terkait: [M9 — Nomination Status](m9-nomination-status.md), [M10 — Customer Dashboard](m10-customer-dashboard.md)
