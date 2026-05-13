# M9b — EPB (Customer Portal): Functional Requirements

Derived from BRD Section 3.4.9b and Section 2.1 Module 9b (BRD v3.3).

> **Scope note (v3.3):** Sejak v3.3, scope M9b adalah **EPB only**. Bagian Invoice (settlement) dipindahkan ke [M9c — Invoice](../invoice/requirements.md). M9b mengelola siklus pembayaran EPB termasuk partial payment (min 1 USD ekuivalen IDR).

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-EI-01 | Sistem harus menyediakan tab "EPB" di menu "EPB & Invoice" Customer Portal yang menampilkan daftar seluruh EPB customer beserta status pembayaran terkini (Belum Dibayar, Menunggu Verifikasi, Pembayaran Ditolak, Lunas) | Customer | Must Have |
| FR-EI-02 | Customer harus dapat melihat detail EPB (view-only) dengan mengklik item di daftar EPB | Customer | Must Have |
| FR-EI-03 | Untuk EPB berstatus **Belum Dibayar**, customer harus dapat mengklik tombol "Bayar", **menginput nominal pembayaran (minimum 1 USD ekuivalen IDR)**, mengupload bukti pembayaran, dan melakukan Submit | Customer | Must Have |
| FR-EI-04 | Untuk EPB berstatus **Menunggu Verifikasi**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer | Must Have |
| FR-EI-05 | Untuk EPB berstatus **Pembayaran Ditolak**, customer harus dapat mengklik "Revisi Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer | Must Have |
| FR-EI-06 | Untuk EPB berstatus **Lunas**, sistem harus menampilkan detail dalam mode view-only sebagai konfirmasi bahwa pembayaran telah selesai | Customer | Must Have |
| FR-EI-07 | Sistem harus menerima status update pembayaran dari STS Platform dan memperbarui tampilan secara real-time. Event yang diterima: `EPB_PAYMENT_REJECT`, `EPB_PAID`, `EPB_SHORTFALL_DETECTED` | System | Must Have |
| FR-EI-08 | Sistem harus mengirimkan data bukti pembayaran beserta nominal yang dibayarkan (`paid_amount`) ke STS Platform via API untuk proses verifikasi | System | Must Have |
| FR-EI-09 | Setelah customer upload proof (FR-EI-03 atau FR-EI-05), sistem harus mengupdate `nominations.status` menjadi `WAITING_PAYMENT_VERIFICATION` secara bersamaan dengan `epb_payments.status` dalam satu transaksi DB | System | Must Have |
| FR-EI-10 | Customer harus dapat membayar EPB secara parsial (paid_amount < total_amount), dengan nominal minimum **1 USD ekuivalen IDR**. Saat EPB Lunas dengan paid_amount < total_amount, STS mengirim webhook `EPB_SHORTFALL_DETECTED` yang men-trigger pembuatan record Invoice di M9c dengan source `EPB_SHORTFALL` dan amount = shortfall | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Webhook dari STS untuk payment status update. Latency pengiriman payment proof + paid_amount ke STS max 3 detik. Retry max 3× exponential backoff. |
| Real-time | BRD §3.5 #2 | Status payment harus tampil ke customer dalam < 1 menit setelah STS mengirim webhook. |
| Auditability | BRD §3.5 #3 | Setiap perubahan status payment + setiap payment attempt (nominal, timestamp, actor) ter-log di audit log. |
| Data Retention | BRD §3.5 #4 | Data EPB payment disimpan minimal 5 tahun. |
| Currency Conversion | New v3.3 | Validasi min 1 USD dilakukan dengan kurs USD→IDR yang di-cache dari STS (refresh harian). Jika kurs tidak tersedia, default ke threshold IDR fixed (ditentukan oleh System Config — `MIN_PAYMENT_IDR_FALLBACK`). |
