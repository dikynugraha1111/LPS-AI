# M9c — Invoice (Customer Portal): Functional Requirements

Derived from BRD Section 3.4.9c and Section 2.1 Module 9c (BRD v3.3).

> **Modul baru di v3.3.** Hasil pemisahan EPB & Invoice menjadi dua modul terpisah. M9c menangani **Invoice** (tagihan settlement); M9b menangani **EPB** (tagihan awal).

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-IN-01 | Sistem harus menyediakan tab "Invoice" di menu "EPB & Invoice" Customer Portal yang menampilkan daftar seluruh Invoice customer beserta status pembayaran terkini (Belum Dibayar, Menunggu Verifikasi, Pembayaran Ditolak, Lunas) | Customer | Must Have |
| FR-IN-02 | Sistem harus menerima webhook dari STS Platform yang men-trigger pembuatan Invoice. Source webhook ada dua: (a) `EPB_SHORTFALL_DETECTED` — dengan referensi ke parent EPB; (b) `ADDITIONAL_SERVICE_INVOICE` — dengan daftar service keys yang ditagihkan. Sistem harus menyimpan source, parent reference, dan amount | System | Must Have |
| FR-IN-03 | Customer harus dapat melihat detail Invoice (view-only) dengan mengklik item di daftar Invoice; detail menampilkan source (Shortfall EPB / Additional Service), referensi parent (jika EPB shortfall) atau daftar service yang ditagih (jika additional service), amount, jatuh tempo | Customer | Must Have |
| FR-IN-04 | Untuk Invoice berstatus **Belum Dibayar**, customer harus dapat memulai proses pembayaran dengan mengklik tombol "Bayar", mengupload bukti pembayaran, dan melakukan Submit | Customer | Must Have |
| FR-IN-05 | Untuk Invoice berstatus **Menunggu Verifikasi**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer | Must Have |
| FR-IN-06 | Untuk Invoice berstatus **Pembayaran Ditolak**, customer harus dapat mengklik "Revisi Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer | Must Have |
| FR-IN-07 | Sistem harus menerima status update pembayaran Invoice dari STS Platform. STS mengirim event: `INVOICE_PAYMENT_REJECT` dan `INVOICE_PAID`. Status Menunggu Verifikasi di-set langsung oleh LPS saat customer upload proof | System | Must Have |
| FR-IN-08 | Invoice **tidak memblokir** lifecycle nominasi. Status nominasi `PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (M9b), bukan oleh Invoice. Invoice adalah settlement separate yang dapat berjalan paralel/setelah voyage | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Webhook dari STS untuk creation + status update. Latency pengiriman payment proof ke STS max 3 detik. Retry max 3× exponential backoff. |
| Real-time | BRD §3.5 #2 | Status payment Invoice harus tampil ke customer dalam < 1 menit setelah STS mengirim webhook. |
| Auditability | BRD §3.5 #3 | Setiap perubahan status Invoice + setiap creation event ter-log di audit log. |
| Data Retention | BRD §3.5 #4 | Data Invoice payment disimpan minimal 5 tahun. |
| Idempotency | New v3.3 | Webhook creation (`EPB_SHORTFALL_DETECTED`, `ADDITIONAL_SERVICE_INVOICE`) harus idempotent — duplicate webhook tidak boleh membuat row baru. Idempotency key: kombinasi `lps_nomination_id` + `source` + `source_ref` (parent_epb_id atau service_keys hash). |
