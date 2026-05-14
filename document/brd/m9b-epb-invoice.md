# BRD Modul M9b — EPB (Customer Portal)

**Kategori:** Customer Portal  
**Versi BRD:** 3.5  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m9b-epb-invoice.md`](../../implementation/replit-handoff/m9b-epb-invoice.md)  
**Sumber:** [BRD Utama — Section 2.1 (9b) & 3.4.9b](../BRD-LPS-V3-Analysis.md)

> **Catatan v3.5:** Halaman Detail EPB diperluas dengan section **"Detail Nominasi"** yang selalu tampil — menampilkan data induk nominasi (Kapal, Tipe Kapal, Jenis Cargo, Towage Plan, ETA, Agen, Charterer, Dibuat, Diperbarui) dari JOIN ke tabel `nominations`. Spesifikasi Payment Instruction Box (Block 3) dipertegas: 2-column grid di dalam box amber, Total di-highlight bold + navy. Tambah FR-EI-13.

> **Catatan v3.4:** Detail Tagihan EPB diperluas menjadi **invoice-style display** — vessel operasional info, line items breakdown (volume/rate/jumlah), kalkulasi subtotal/PPn 11%/total, dan instruksi pembayaran lengkap (bank, no rekening, atas nama, kode bayar, batas pembayaran). Tambah aksi **Download EPB PDF**. Currency support multi-currency (IDR / USD). Sumber data: STS webhook APPROVED dengan field tambahan (`vessel_ops`, `line_items[]`, `subtotal`, `vat_amount`, `bank_info`, `kode_bayar`, `epb_pdf_url`). Tambah FR-EI-11 dan FR-EI-12.

> **Catatan v3.3:** M9b sebelumnya bernama "EPB & Invoice" dengan scope gabungan. Mulai BRD v3.3, scope M9b adalah **EPB only**. Bagian Invoice dipisah menjadi modul baru: [M9c — Invoice](m9c-invoice.md). UI keduanya tetap dalam satu menu "EPB & Invoice" di sidebar Customer Portal, namun dengan tab internal terpisah (EPB / Invoice). Backend dan struktur module dipisah penuh.

---

## Definisi

Modul dalam Customer Portal LPS yang mengelola pembayaran **EPB (Estimasi Perkiraan Biaya)** — tagihan awal yang dikeluarkan STS Platform setelah nominasi disetujui. Customer dapat membayar penuh atau parsial (minimum 1 USD ekuivalen IDR); shortfall otomatis dipindahkan ke modul Invoice (M9c) sebagai tagihan settlement. Data EPB di-generate oleh STS Platform; LPS hanya menampilkan dan mengelola proses upload bukti bayar.

## Tujuan

- Menyediakan visibilitas status payment EPB bagi customer.
- Memfasilitasi partial payment EPB (min 1 USD) dengan auto-routing shortfall ke modul Invoice.
- Memfasilitasi upload ulang bukti pembayaran jika sebelumnya ditolak.

## Cakupan (In Scope)

### 4 Status Payment EPB

| Status | Deskripsi | Aksi Customer |
|--------|-----------|---------------|
| **Belum Dibayar** | Record EPB dibuat otomatis saat nominasi di-approve STS. | Click "Bayar" → Input nominal (≥ 1 USD) → Upload Proof → Submit |
| **Menunggu Verifikasi** | Bukti sudah disubmit, menunggu verifikasi STS Platform. | View-only |
| **Pembayaran Ditolak** | Bukti ditolak STS Platform. | Click "Revisi Data" → Upload ulang → Submit |
| **Lunas** | Pembayaran dikonfirmasi STS Platform. | View-only. Jika `paid_amount < total_amount` → STS push Invoice (M9c). |

### Flow Per Status

```
Belum Dibayar
  └─ Click Bayar → Input nominal (≥ 1 USD) → Upload Payment Proof → Submit
       └─ Menunggu Verifikasi (view-only)
            ├─ Pembayaran Ditolak → Click Revisi Data → Upload ulang → Submit → Menunggu Verifikasi
            └─ Lunas (view-only, selesai)
                  └─ Jika paid_amount < total_amount → STS webhook EPB_SHORTFALL_DETECTED → M9c Invoice dibuat
```

### Partial Payment Logic

- Customer bebas memilih nominal pembayaran selama **≥ 1 USD ekuivalen IDR** (validasi LPS sebelum kirim ke STS).
- STS Platform menentukan apakah pembayaran tersebut diterima (Lunas) atau ditolak.
- Saat status EPB = Lunas dengan `paid_amount < total_amount`, STS Platform mengirim webhook `EPB_SHORTFALL_DETECTED` yang men-trigger pembuatan record Invoice di M9c dengan source `EPB_SHORTFALL` dan amount = shortfall.

### Data EPB yang Ditampilkan (Invoice-style, v3.4)

Detail Tagihan EPB menampilkan field berikut, seluruhnya bersumber dari STS Platform (read-only):

| Kelompok | Field |
|---|---|
| **Identitas** | Nomor EPB, No. Nominasi |
| **Operasional Voyage** | Vessel, Crane, STS Slot, Mooring Team, ETA, Surveyor, Anchor, Est. Duration |
| **Line Items** | Item Layanan, Volume, Rate, Jumlah (minimal satu baris STS Fee; baris tambahan per service key Additional Service) |
| **Kalkulasi** | Subtotal, PPn 11%, Total |
| **Instruksi Pembayaran** | Bank, No Rekening, Atas Nama, Kode Bayar, Batas Pembayaran (dengan indikator sisa hari) |
| **Dokumen** | URL PDF EPB resmi (`epb_pdf_url`) |

**Currency support:** EPB dapat diterbitkan dalam IDR atau USD (ditentukan STS per nominasi). UI render sesuai field `currency` dari payload. Validasi minimum partial payment tetap 1 USD ekuivalen (pakai `USD_IDR_RATE` saat currency = IDR).

### Aksi Customer di Detail EPB

- **Download EPB PDF:** tersedia di semua status. File di-stream LPS dari STS via proxy endpoint (LPS tidak generate PDF).
- **Bayar / Revisi Data:** kontextual sesuai status (lihat flow di atas).

### STS Webhook yang Ditangani M9b

| Event | Aksi LPS |
|-------|----------|
| `EPB_PAYMENT_REJECT` | Update `epb_payments.status` → Pembayaran Ditolak |
| `EPB_PAID` | Update `epb_payments.status` → Lunas. Update `nominations.status` → PAYMENT_CONFIRMED dalam satu transaksi. |
| `EPB_SHORTFALL_DETECTED` | Forward ke handler M9c untuk create Invoice row. |

> Status Menunggu Verifikasi di-set langsung oleh LPS saat customer upload proof — tidak perlu webhook dari STS.

## Batasan Modul

- Verifikasi pembayaran (keputusan Reject/Paid) sepenuhnya dilakukan oleh STS Platform.
- LPS hanya menerima status update dari STS Platform dan menampilkannya ke customer.
- Satu endpoint upload melayani dua flow (Bayar dari Belum Dibayar dan Revisi Data dari Pembayaran Ditolak) — dibedakan oleh status saat ini.
- Setelah customer upload di M9b, status nominasi di M9 juga ikut diupdate dalam satu transaksi DB.
- **Invoice (settlement) bukan tanggung jawab M9b** — dikelola sepenuhnya di [M9c — Invoice](m9c-invoice.md).

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-EI-01 | Sistem harus menyediakan tab "EPB" di menu "EPB & Invoice" Customer Portal yang menampilkan daftar seluruh EPB customer beserta status pembayaran terkini (Belum Dibayar, Menunggu Verifikasi, Pembayaran Ditolak, Lunas) | Customer |
| FR-EI-02 | Customer harus dapat melihat detail EPB (view-only) dengan mengklik item di daftar EPB | Customer |
| FR-EI-03 | Untuk EPB berstatus **Belum Dibayar**, customer harus dapat memulai proses pembayaran dengan mengklik tombol "Bayar", **menginput nominal pembayaran (minimum 1 USD ekuivalen IDR)**, mengupload bukti pembayaran, dan melakukan Submit | Customer |
| FR-EI-04 | Untuk EPB berstatus **Menunggu Verifikasi**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer |
| FR-EI-05 | Untuk EPB berstatus **Pembayaran Ditolak**, customer harus dapat mengklik "Revisi Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer |
| FR-EI-06 | Untuk EPB berstatus **Lunas**, sistem harus menampilkan detail dalam mode view-only sebagai konfirmasi bahwa pembayaran telah selesai | Customer |
| FR-EI-07 | Sistem harus menerima status update pembayaran dari STS Platform dan memperbarui tampilan di tab EPB secara real-time. STS mengirim event: `EPB_PAYMENT_REJECT` dan `EPB_PAID`. Status Menunggu Verifikasi di-set langsung oleh LPS saat customer upload proof | System |
| FR-EI-08 | Sistem harus mengirimkan data bukti pembayaran beserta nominal yang dibayarkan (`paid_amount`) ke STS Platform via API untuk proses verifikasi | System |
| FR-EI-09 | Setelah customer berhasil upload bukti pembayaran, sistem harus mengupdate status nominasi terkait di M9 menjadi "Waiting Payment Verification" secara bersamaan dalam satu transaksi | System |
| FR-EI-10 | Customer harus dapat membayar EPB secara parsial (paid_amount < total_amount), dengan nominal minimum **1 USD ekuivalen IDR**. Saat status EPB berubah ke Lunas dengan paid_amount < total_amount, STS Platform mengirim webhook `EPB_SHORTFALL_DETECTED` yang men-trigger pembuatan record Invoice di M9c dengan source `EPB_SHORTFALL` dan amount = shortfall | System |
| FR-EI-11 | Halaman Detail EPB harus menampilkan **invoice-style detail tagihan** mencakup: (a) data operasional voyage (Vessel, Crane, STS Slot, Mooring Team, ETA, Surveyor, Anchor, Est. Duration); (b) line items table (Item Layanan, Volume, Rate, Jumlah) dengan Subtotal + PPn 11% + Total (currency mengikuti field `currency` dari STS); (c) instruksi pembayaran (Bank, No Rekening, Atas Nama, Kode Bayar, Batas Pembayaran dengan indikator sisa hari, warning bila ≤ 3 hari). Seluruh data read-only dari STS via webhook APPROVED | Customer |
| FR-EI-12 | Customer harus dapat mengunduh dokumen EPB resmi (PDF) melalui tombol **"Download EPB PDF"** yang tersedia di semua status. File di-stream LPS dari `epb_pdf_url` STS via proxy endpoint; LPS tidak generate PDF mandiri | Customer |
| FR-EI-13 | Halaman Detail EPB harus menampilkan section **"Detail Nominasi"** yang selalu tampil di semua status, berisi data nominasi induk berikut: Kapal (vessel_name), Tipe Kapal (vessel_type), Jenis Cargo (cargo_type), Towage Plan (towage_plan), ETA, Agen (agent_name), Charterer, Dibuat (created_at nominasi), Diperbarui (updated_at nominasi). Data bersumber dari JOIN `nominations` di endpoint `GET /api/customer/epb-payments/:id` — bukan data duplikasi | Customer |

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

- BRD Utama: [Section 2.1 (9b)](../BRD-LPS-V3-Analysis.md) & [Section 3.4.9b](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/epb-invoice/`](../../module/epb-invoice/)
- Replit handoff: [`m9b-epb-invoice.md`](../../implementation/replit-handoff/m9b-epb-invoice.md)
- Modul terkait:
  - [M9 — Nomination Status](m9-nomination-status.md) (parent EPB)
  - [M9c — Invoice](m9c-invoice.md) (settlement shortfall + additional service)
  - [M10 — Customer Dashboard](m10-customer-dashboard.md)
