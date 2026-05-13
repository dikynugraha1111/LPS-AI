# BRD Modul M9c — Invoice (Customer Portal)

**Kategori:** Customer Portal  
**Versi BRD:** 3.3 (diperkenalkan)  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m9c-invoice.md`](../../implementation/replit-handoff/m9c-invoice.md)  
**Sumber:** [BRD Utama — Section 2.1 (9c) & 3.4.9c](../BRD-LPS-V3-Analysis.md)

> **Catatan v3.3:** Modul M9c diperkenalkan di BRD v3.3 sebagai hasil pemisahan modul EPB & Invoice menjadi dua modul terpisah. M9c menangani **Invoice** (tagihan settlement). M9b menangani **EPB** (tagihan awal). UI keduanya tetap berada dalam satu menu "EPB & Invoice" di sidebar Customer Portal dengan tab internal terpisah.

---

## Definisi

Modul dalam Customer Portal LPS yang mengelola pembayaran **Invoice** — tagihan settlement yang muncul dari (a) shortfall pembayaran EPB sebelumnya atau (b) Additional Service yang dipakai customer selama voyage. Invoice di-generate otomatis oleh STS Platform via webhook dan dikelola di LPS sebagai interface customer.

## Tujuan

- Menyediakan visibilitas tagihan settlement (shortfall EPB + Additional Service) bagi customer.
- Memfasilitasi upload bukti pembayaran Invoice dan re-upload jika ditolak.
- Memisahkan secara backend antara tagihan utama (EPB) dan tagihan settlement (Invoice) sesuai realita business flow.

## Cakupan (In Scope)

### Source Invoice

| Source | Trigger | Webhook Event | Konten |
|--------|---------|---------------|--------|
| **EPB_SHORTFALL** | Customer membayar EPB parsial (`paid_amount < total_amount`) dan STS confirm EPB Lunas | `EPB_SHORTFALL_DETECTED` | `parent_epb_id`, `amount = shortfall` |
| **ADDITIONAL_SERVICE** | Customer pakai Additional Service selama voyage (Tank Cleaning, Bunkering & Fresh Water Supplying, Short Stay Temporary, Supply Logistic, Lay Up, Ship Chandler, Kapal Emergency) | `ADDITIONAL_SERVICE_INVOICE` | `service_keys[]`, `amount` |

### 4 Status Payment Invoice

| Status | Deskripsi | Aksi Customer |
|--------|-----------|---------------|
| **Belum Dibayar** | Record Invoice dibuat otomatis oleh webhook STS. | Click "Bayar" → Upload Proof → Submit |
| **Menunggu Verifikasi** | Bukti sudah disubmit, menunggu verifikasi STS Platform. | View-only |
| **Pembayaran Ditolak** | Bukti ditolak STS Platform. | Click "Revisi Data" → Upload ulang → Submit |
| **Lunas** | Pembayaran dikonfirmasi STS Platform. | View-only (terminal) |

### Flow Per Status

```
Belum Dibayar
  └─ Click Bayar → Upload Payment Proof → Submit
       └─ Menunggu Verifikasi (view-only)
            ├─ Pembayaran Ditolak → Click Revisi Data → Upload ulang → Submit → Menunggu Verifikasi
            └─ Lunas (view-only, terminal)
```

### STS Webhook yang Ditangani M9c

| Event | Aksi LPS |
|-------|----------|
| `EPB_SHORTFALL_DETECTED` | Create row di tabel `invoices` dengan `source = EPB_SHORTFALL`, `parent_epb_id`, `amount`, `status = Belum Dibayar` |
| `ADDITIONAL_SERVICE_INVOICE` | Create row di tabel `invoices` dengan `source = ADDITIONAL_SERVICE`, `service_keys[]`, `amount`, `status = Belum Dibayar` |
| `INVOICE_PAYMENT_REJECT` | Update `invoices.status` → Pembayaran Ditolak |
| `INVOICE_PAID` | Update `invoices.status` → Lunas |

> Status Menunggu Verifikasi di-set langsung oleh LPS saat customer upload proof — tidak perlu webhook dari STS.

## Batasan Modul

- Verifikasi pembayaran Invoice sepenuhnya dilakukan oleh STS Platform.
- LPS **tidak** generate Invoice secara mandiri — semua Invoice berasal dari webhook STS.
- Satu endpoint upload melayani dua flow (Bayar dari Belum Dibayar dan Revisi Data dari Pembayaran Ditolak).
- **Invoice tidak memblokir lifecycle nominasi.** Nominasi `PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (M9b), bukan oleh Invoice. Invoice adalah settlement separate.
- Generate EPB (parent) bukan tanggung jawab M9c — itu di [M9b — EPB](m9b-epb-invoice.md).

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-IN-01 | Sistem harus menyediakan tab "Invoice" di menu "EPB & Invoice" Customer Portal yang menampilkan daftar seluruh Invoice customer beserta status pembayaran terkini (Belum Dibayar, Menunggu Verifikasi, Pembayaran Ditolak, Lunas) | Customer |
| FR-IN-02 | Sistem harus menerima webhook dari STS Platform yang men-trigger pembuatan Invoice. Source webhook ada dua: (a) `EPB_SHORTFALL_DETECTED` — dengan referensi ke parent EPB; (b) `ADDITIONAL_SERVICE_INVOICE` — dengan daftar service keys yang ditagihkan. Sistem harus menyimpan source, parent reference, dan amount | System |
| FR-IN-03 | Customer harus dapat melihat detail Invoice (view-only) dengan mengklik item di daftar Invoice; detail menampilkan source (Shortfall EPB / Additional Service), referensi parent (jika EPB shortfall) atau daftar service yang ditagih (jika additional service), amount, jatuh tempo | Customer |
| FR-IN-04 | Untuk Invoice berstatus **Belum Dibayar**, customer harus dapat memulai proses pembayaran dengan mengklik tombol "Bayar", mengupload bukti pembayaran, dan melakukan Submit | Customer |
| FR-IN-05 | Untuk Invoice berstatus **Menunggu Verifikasi**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer |
| FR-IN-06 | Untuk Invoice berstatus **Pembayaran Ditolak**, customer harus dapat mengklik "Revisi Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer |
| FR-IN-07 | Sistem harus menerima status update pembayaran Invoice dari STS Platform. STS mengirim event: `INVOICE_PAYMENT_REJECT` dan `INVOICE_PAID`. Status Menunggu Verifikasi di-set langsung oleh LPS saat customer upload proof | System |
| FR-IN-08 | Invoice **tidak memblokir** lifecycle nominasi. Status nominasi `PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (M9b), bukan oleh Invoice. Invoice adalah settlement separate yang dapat berjalan paralel/setelah voyage | System |

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

- BRD Utama: [Section 2.1 (9c)](../BRD-LPS-V3-Analysis.md) & [Section 3.4.9c](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/invoice/`](../../module/invoice/)
- Replit handoff: [`m9c-invoice.md`](../../implementation/replit-handoff/m9c-invoice.md)
- Modul terkait:
  - [M9b — EPB](m9b-epb-invoice.md) (parent source untuk shortfall)
  - [M8 — Nomination Submission](m8-nomination-submission.md) (parent source untuk Additional Service)
  - [M9 — Nomination Status](m9-nomination-status.md)
  - [M10 — Customer Dashboard](m10-customer-dashboard.md)
