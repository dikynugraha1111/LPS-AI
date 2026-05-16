# M13 — Mother Vessel Master (Customer Portal): Functional Requirements

Derived from BRD Section 3.4.13 and Section 2.1 Module 13 (BRD v3.5, change log #29).

> **Scope note v1.0:** Phase 1 — **MV only**. Halaman khusus Mother Vessel; tidak ada tabs/filter/list untuk asset type lain. Data master MV otoritatif di STS Platform; LPS sebagai UI proxy.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-MV-01 | Sistem harus menyediakan menu **"Mother Vessel"** di sidebar Customer Portal yang menampilkan daftar Mother Vessel (MV) milik organisasi customer (data master di-fetch dari STS Platform via API) | Customer | Must Have |
| FR-MV-02 | Halaman Mother Vessel harus menampilkan **list MV milik organisasi customer** dengan search (asset code, vessel name, IMO, owner) dan pagination. Phase 1 **khusus MV** — tidak ada tabs filter asset type lain | Customer | Must Have |
| FR-MV-03 | Customer harus dapat menambahkan MV baru melalui tombol **"Add New MV"** yang membuka form dengan field: Asset Code (auto-generate saat save), Vessel Name, IMO Number, Type, Call Sign, Classification, Capacity (DWT), GRT, Length LOA (m), Beam (m), Max Draft (m), Owner (auto-set ke organisasi customer, read-only), Status. Submit form akan membuat **approval request** ke STS Platform via API | Customer | Must Have |
| FR-MV-04 | Customer harus dapat mengedit MV existing miliknya melalui aksi Edit (icon pencil) yang membuka form yang sama dengan data ter-prefill. Submit edit akan membuat **approval request** ke STS Platform via API | Customer | Must Have |
| FR-MV-05 | Customer harus dapat me-non-aktifkan (Deactivate) MV miliknya melalui aksi Deactivate (icon power). Aksi ini akan membuat **approval request** ke STS Platform untuk mengubah `status` MV menjadi `INACTIVE` | Customer | Must Have |
| FR-MV-06 | Sistem harus menampilkan **disclaimer banner** di setiap form Add/Edit/Deactivate dengan copy: "All changes will be submitted as an approval request and require admin review before being applied." | Customer | Must Have |
| FR-MV-07 | Sistem harus mengirim seluruh aksi customer (Add / Edit / Deactivate MV) sebagai **approval request** ke STS Platform via API. LPS **tidak menyimpan tabel master MV** — data otoritatif tetap di STS. LPS hanya menyimpan cache status approval request dengan TTL singkat (default 600 detik) | System | Must Have |
| FR-MV-08 | Sistem harus menampilkan **status approval** per MV di tabel list dengan badge yang sesuai: `ACTIVE` (Confirmed hijau), `INACTIVE` (Neutral abu-abu), `PENDING APPROVAL` (Pending verify kuning), `REJECTED` (Error merah). Status diambil dari STS API | Customer | Must Have |
| FR-MV-09 | Record MV dengan status `PENDING APPROVAL` **tetap tampil** di tabel Mother Vessel, namun **tidak boleh dipilih** sebagai MV di form Nomination Request (M8). Validasi terjadi di sisi LPS — endpoint vessel dropdown M8 hanya mereturn MV ber-status `ACTIVE` | System | Must Have |
| FR-MV-10 | Untuk MV ber-status `PENDING APPROVAL`, tombol **Edit** dan **Deactivate** harus **disabled** (lock) sampai STS Admin selesai verifikasi (approve atau reject). Tooltip: "Edit tidak tersedia — sedang menunggu approval admin." Mencegah race condition multiple approval request | System | Must Have |
| FR-MV-11 | Sistem harus menyediakan view detail MV (icon mata) yang menampilkan section **"Riwayat Perubahan"** berisi daftar approval request untuk MV tersebut: timestamp submit, jenis aksi (Add/Edit/Deactivate), user yang submit, status (Pending/Approved/Rejected), reviewer (STS Admin), alasan reject (jika ada), timestamp resolution. Data di-fetch dari STS API | Customer | Must Have |
| FR-MV-12 | Sistem harus menampilkan **alasan reject** dari STS Admin pada MV ber-status `REJECTED` (hover state pada badge atau di section Riwayat Perubahan), agar customer mengetahui apa yang perlu diperbaiki sebelum re-submit | Customer | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| STS Integration | BRD §3.5 #9 | Semua API call ke STS (list, detail, submit approval request, fetch history) max latency 3 detik per request. Retry max 3× exponential backoff untuk POST submission. GET listing pakai cache TTL 600s. |
| Real-time | BRD §3.5 #2 | Status approval result (approved/rejected) harus tampil ke customer dalam < 1 menit setelah STS Admin verifikasi (cache invalidation atau polling). |
| Auditability | BRD §3.5 #3 | Setiap aksi customer (Add/Edit/Deactivate submit) harus ter-log di audit log LPS (siapa, kapan, payload diff). Audit STS-side (approval result) hanya display via API STS. |
| Security | BRD §3.5 #5 | Validasi `owner` server-side: customer hanya bisa Add/Edit/Deactivate MV dengan `organization_id` yang match JWT. LPS reject 403 jika mismatch sebelum forward ke STS. |
| Fallback | BRD §3.5 #6 | Jika STS API unreachable: tampilkan banner "Layanan master vessel sedang tidak tersedia. Silakan coba lagi." dan disable tombol Add/Edit/Deactivate. Cache list view tetap tampil (read-only) dengan indikator "Data terakhir dimuat: {timestamp}". |
| Data Ownership | New v1.0 | LPS **tidak menyimpan** tabel `mother_vessels`. Semua data master via proxy ke STS. Cache lokal hanya untuk status approval request (TTL 600s) — bukan untuk field master MV. |
