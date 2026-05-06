# M9 — Nomination Status & Payment: Functional Requirements

Derived from BRD Section 3.4.9 and Section 2.1 Module 9.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-NP-01 | Sistem harus menerima status update nominasi dari STS Platform (Pending, Approved, Need Revision) dan menampilkannya ke customer. Status Pending ditampilkan dengan label "Menunggu proses di STS Platform". | System | Must Have |
| FR-NP-02 | Customer harus dapat melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta detail nominal yang harus dibayarkan | Customer | Must Have |
| FR-NP-03 | Customer harus dapat melihat detail jadwal (anchor point, ETB, estimasi durasi) setelah nominasi di-approve oleh STS Platform | Customer | Must Have |
| FR-NP-04 | Sistem harus mengirim notifikasi ke customer saat status nominasi berubah | System | Must Have |
| FR-NP-05 | Customer harus dapat melakukan revisi data nominasi jika status = Need Revision, kemudian re-submit | Customer | Must Have |
| FR-NP-06 | Customer harus dapat mengupload bukti pembayaran (Proof of Payment) melalui LPS Portal sesuai dengan EPB yang diterima dari STS Platform. Format yang didukung: PDF, JPG, PNG (max 5MB) | Customer | Must Have |
| FR-NP-07 | Sistem harus mengirimkan data bukti pembayaran yang diupload customer ke STS Platform via API untuk proses verifikasi | System | Must Have |
| FR-NP-08 | Setelah customer mengupload bukti pembayaran, status nominasi harus berubah menjadi "Waiting Payment Verification" | System | Must Have |
| FR-NP-09 | Sistem harus menerima hasil verifikasi pembayaran dari STS Platform (Confirmed/Rejected) dan menampilkannya ke customer | System | Must Have |
| FR-NP-10 | Jika pembayaran ditolak oleh STS Platform, customer harus menerima notifikasi beserta alasan penolakan dan dapat mengupload ulang bukti pembayaran yang benar | Customer | Must Have |
| FR-NP-11 | Setelah pembayaran dikonfirmasi oleh STS Platform, status nominasi berubah menjadi "Payment Confirmed" dan voyage dapat dimulai | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Webhook/callback dari STS untuk status update. LPS menerima dan menyimpan. Latency pengiriman payment proof max 3 detik. |
| Real-time | BRD §3.5 #2 | Status change harus tampil ke customer dalam < 1 menit setelah STS mengirim update. |
| Auditability | BRD §3.5 #3 | Setiap perubahan status nominasi ter-timestamp dan masuk ke audit log. |
| Notification | BRD §3.4.9 FR-NP-04 | Notifikasi in-app dan/atau email pada setiap perubahan status. |
