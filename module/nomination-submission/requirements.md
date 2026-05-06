# M8 — Nomination Request Submission: Functional Requirements

Derived from BRD Section 3.4.8 and Section 2.1 Module 8.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-NS-01 | Sistem harus menyediakan form pengajuan nominasi dengan field: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Estimasi Jumlah Barge | Customer | Must Have |
| FR-NS-02 | Customer harus dapat mengupload dokumen pendukung nominasi: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, Nomor PKK (input manual) | Customer | Must Have |
| FR-NS-03 | Customer harus dapat menyimpan nominasi sebagai Draft sebelum melakukan Submit | Customer | Must Have |
| FR-NS-04 | Sistem harus mengenerate Nomor Nominasi otomatis setelah customer melakukan Submit | System | Must Have |
| FR-NS-05 | Sistem harus mengirimkan data nominasi ke STS Platform via API setelah customer melakukan Submit | System | Must Have |

## Cross-Cutting Requirements Applicable to This Module

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | API call ke STS max latency 3 detik. Retry max 3× exponential backoff jika gagal. |
| Auditability | BRD §3.5 #3 | Setiap submit nomination harus ter-timestamp dan masuk ke audit log. |
| Data Retention | BRD §3.5 #4 | Data nominasi (termasuk Draft) disimpan minimal 5 tahun. |
