# M9b — EPB & Invoice (Customer Portal): Functional Requirements

Derived from BRD Section 3.4.9b and Section 2.1 Module 9b (BRD v3.1).

> **Scope note:** M9b handles the payment verification cycle after the first proof of payment has been submitted via M9. The entry point (EPB Confirmation) belongs to M9.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-EI-01 | Sistem harus menyediakan menu EPB & Invoice di Customer Portal yang menampilkan daftar seluruh EPB customer beserta status pembayaran terkini (Unpaid, Pending Review, Payment Reject, Paid) | Customer | Must Have |
| FR-EI-02 | Customer harus dapat melihat detail EPB dan Invoice (view-only) dengan mengklik item di daftar EPB & Invoice | Customer | Must Have |
| FR-EI-03 | Untuk EPB berstatus **Unpaid**, customer harus dapat memulai proses pembayaran dengan mengklik tombol "Pay", kemudian mengupload bukti pembayaran dan mengisi data terkait, kemudian Submit | Customer | Must Have |
| FR-EI-04 | Untuk EPB berstatus **Pending Review**, sistem harus menampilkan status dalam mode view-only; tidak ada aksi yang dapat dilakukan customer hingga STS Platform memberikan keputusan | Customer | Must Have |
| FR-EI-05 | Untuk EPB berstatus **Payment Reject**, customer harus dapat mengklik "Revision Data" untuk mengupload ulang bukti pembayaran yang benar dan melakukan re-submit | Customer | Must Have |
| FR-EI-06 | Untuk EPB berstatus **Paid**, sistem harus menampilkan detail dalam mode view-only sebagai konfirmasi bahwa pembayaran telah selesai | Customer | Must Have |
| FR-EI-07 | Sistem harus menerima status update pembayaran dari STS Platform (Pending Review → Payment Reject / Paid) dan memperbarui tampilan di menu EPB & Invoice secara real-time | System | Must Have |
| FR-EI-08 | Sistem harus mengirimkan data bukti pembayaran (upload dari FR-EI-03 atau FR-EI-05) ke STS Platform via API untuk proses verifikasi | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Webhook dari STS untuk payment status update. Latency pengiriman payment proof ke STS max 3 detik. Retry max 3× exponential backoff. |
| Real-time | BRD §3.5 #2 | Status payment harus tampil ke customer dalam < 1 menit setelah STS mengirim webhook. |
| Auditability | BRD §3.5 #3 | Setiap perubahan status payment ter-timestamp dan masuk ke audit log. |
| Data Retention | BRD §3.5 #4 | Data EPB payment disimpan minimal 5 tahun. |
