# M7 — Customer Authentication & Onboarding: Functional Requirements

Derived from BRD Section 3.4.7 and Section 2.1 Module 7.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-CA-01 | Sistem harus menyediakan halaman registrasi bagi customer baru dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note; Tipe Pelanggan di-default ke "Cargo Owner" oleh sistem dan tidak ditampilkan di form | Customer | Must Have |
| FR-CA-02 | Customer wajib mengupload 3 dokumen saat registrasi: NPWP (file), NIP (file), Company Profile (file), dengan field tambahan Description, Issue Date, dan Expiry Date opsional per dokumen | Customer | Must Have |
| FR-CA-03 | Customer Code harus di-generate otomatis oleh sistem setelah Admin menyetujui registrasi; tidak diisi oleh customer | System | Must Have |
| FR-CA-04 | Admin harus dapat mengakses halaman Customer Management untuk melihat daftar seluruh customer beserta statusnya (PENDING_VALIDATION, ACTIVE, REJECTED) | Admin | Must Have |
| FR-CA-05 | Admin harus dapat melihat detail data registrasi customer termasuk seluruh dokumen yang diupload; Admin dapat mengunduh dokumen untuk direview | Admin | Must Have |
| FR-CA-06 | Admin harus dapat mengaktifkan (Approve) akun customer berstatus PENDING_VALIDATION; sistem otomatis men-generate Customer Code dan mengirimkan notifikasi aktivasi ke email customer | Admin | Must Have |
| FR-CA-07 | Admin harus dapat menolak (Reject) akun customer berstatus PENDING_VALIDATION; sistem mengirimkan notifikasi penolakan ke email customer | Admin | Must Have |
| FR-CA-08 | Admin harus dapat menambahkan customer baru secara langsung dari Admin Dashboard; akun yang dibuat Admin langsung berstatus ACTIVE dan Customer Code langsung di-generate | Admin | Must Have |
| FR-CA-09 | Saat Admin menambahkan customer secara manual, Admin dapat melampirkan dokumen tambahan (custom documents) selain 3 dokumen standar; setiap custom document dapat diberi nama/label bebas oleh Admin | Admin | Must Have |
| FR-CA-10 | Customer yang sudah terverifikasi (status ACTIVE) harus dapat login dan mengakses dashboard customer | Customer | Must Have |

## Cross-Cutting Requirements Applicable to This Module

| NFR | Source | Description |
|-----|--------|-------------|
| Security | BRD §3.5 #5 | Autentikasi username/password. Password di-hash. Komunikasi HTTPS/TLS. JWT untuk session. |
| Auditability | BRD §3.5 #3 | Seluruh event aktivasi/penolakan akun harus ter-timestamp dan masuk ke immutable audit log. |
| STS Integration | BRD §2.1 M7 | Data customer dikirim ke STS Platform via API setelah Admin mengaktifkan akun. Retry max 3× exponential backoff. |
