# M9 — Nomination Status & EPB Confirmation: Functional Requirements

Derived from BRD Section 3.4.9 and Section 2.1 Module 9 (revised per Swimlane V3, BRD v3.1).

> **Scope boundary:** M9 covers status tracking and the first payment proof submission (EPB Confirmation) only. The payment verification cycle (Unpaid → Pending Review → Payment Reject → Paid) is handled by M9b.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-NP-01 | Sistem harus menerima status update nominasi dari STS Platform (Pending, Approved, Need Revision) dan menampilkannya ke customer. Status Pending ditampilkan dengan label "Menunggu proses di STS Platform". | System | Must Have |
| FR-NP-02 | Jika nominasi berstatus Approved, customer harus dapat melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta detail: schedule, dock, dan nominal yang harus dibayarkan (view-only) | Customer | Must Have |
| FR-NP-03 | Customer harus dapat melihat detail jadwal (schedule, dock) setelah nominasi di-approve oleh STS Platform | Customer | Must Have |
| FR-NP-04 | Sistem harus mengirim notifikasi ke customer saat status nominasi berubah | System | Must Have |
| FR-NP-05 | Customer harus dapat melakukan revisi data nominasi jika status = Need Revision (branch False pada decision node "Is Approved?"), kemudian re-submit ke STS Platform | Customer | Must Have |
| FR-NP-06 | Customer harus dapat mengupload bukti pembayaran pertama (Proof of Payment) beserta data terkait melalui halaman EPB Confirmation di LPS Portal. Format yang didukung: PDF, JPG, PNG (max 5MB) | Customer | Must Have |
| FR-NP-07 | Setelah customer Submit pada halaman EPB Confirmation, data bukti pembayaran dipindahkan ke menu EPB & Invoice (M9b) untuk siklus verifikasi selanjutnya | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Webhook/callback dari STS untuk status update. LPS menerima dan menyimpan. Latency max 3 detik. |
| Real-time | BRD §3.5 #2 | Status change harus tampil ke customer dalam < 1 menit setelah STS mengirim update. |
| Auditability | BRD §3.5 #3 | Setiap perubahan status nominasi ter-timestamp dan masuk ke audit log. |
| Notification | BRD §3.4.9 FR-NP-04 | Notifikasi in-app dan/atau email pada setiap perubahan status. |
