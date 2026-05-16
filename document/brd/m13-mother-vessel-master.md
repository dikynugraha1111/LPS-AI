# BRD Modul M13 — Mother Vessel Master (Customer Portal)

**Kategori:** Customer Portal
**Versi BRD:** 1.0 (BARU)
**Tanggal Update:** Mei 2026
**Status Implementasi:** module/ + UI design + replit handoff (v1.0) tersedia. Contract STS ⚠ TENTATIVE — menunggu konfirmasi tim STS (6 open questions).
**Sumber:** [BRD Utama — Section 2.1 (13) & 3.4.13](../BRD-LPS-V3-Analysis.md)

> **Catatan v1.0 (Mei 2026):** Modul baru hasil briefing customer. Menyediakan kapabilitas pengelolaan master Mother Vessel (MV) di sisi customer dengan workflow approval ke STS Platform. **Data master MV tetap milik STS** — LPS hanya menyediakan UI dan proxy API. Phase 1 scope: **MV only**.

---

## Definisi

Modul dalam Customer Portal LPS yang memungkinkan customer mengelola data master Mother Vessel (MV) miliknya sendiri: Add, Edit, dan Deactivate. Setiap aksi customer menghasilkan **approval request** yang harus di-verifikasi oleh STS Admin sebelum berlaku efektif.

**Prinsip data ownership:** Master MV disimpan di DB STS Platform (single source of truth). LPS **tidak** menyimpan tabel `mother_vessels` di DB lokalnya — seluruh operasi list, detail, dan CRUD dilakukan via proxy ke STS API. LPS hanya boleh menyimpan cache status approval request dengan TTL singkat (default 5–15 menit) untuk performa UI.

## Tujuan

- Memberikan kontrol langsung kepada customer untuk mengelola data master MV miliknya tanpa harus email/ticket ke STS Admin.
- Mempertahankan otoritas data master di STS Platform (single source of truth) — LPS sebagai UI proxy.
- Menyediakan audit trail per MV (riwayat approval request) untuk transparency dan compliance.
- Mencegah race condition: pending request mengunci record sampai STS Admin selesai verifikasi.

## Cakupan (In Scope)

### Customer Capabilities (Phase 1 — MV Only)

| Aksi | Deskripsi | Hasil |
|------|-----------|-------|
| **Add New MV** | Customer mengisi form dengan field: Asset Code (auto-generate), Vessel Name, IMO Number, Type, Call Sign, Classification, Capacity (DWT), GRT, Length LOA, Beam, Max Draft, Owner (auto-set), Status. | Submit → approval request `PENDING` di STS. |
| **Edit MV** | Customer pilih MV existing → ubah field → submit. | Submit → approval request `PENDING` di STS (untuk perubahan). |
| **Deactivate MV** | Customer klik tombol power → konfirmasi → submit. | Submit → approval request `PENDING` di STS (untuk status change → `INACTIVE`). |

### Lifecycle Approval Request

```
Customer submit aksi (Add / Edit / Deactivate)
  └─ POST ke STS API (LPS proxy)
       └─ Approval request created — status PENDING
            ├─ STS Admin Approve → APPROVED
            │     └─ Master MV di STS terupdate
            │     └─ MV dapat dipakai di Nomination (M8)
            └─ STS Admin Reject → REJECTED
                  └─ Master MV tidak berubah
                  └─ Customer lihat alasan reject di Riwayat Perubahan
                  └─ Customer dapat submit ulang (Edit lagi atau Add baru)
```

### List View Customer Portal

- **Sidebar menu:** "Mother Vessel"
- **Header page:** "Mother Vessel" + tombol primary "+ Add New MV"
- **Subtitle:** "View and manage your organization's Mother Vessels. All changes require approval."
- **Search bar:** filter by asset code, vessel name, IMO, atau owner
- **Tabel kolom:**
  - Action icons: View (mata), Edit (pencil), Deactivate (power)
  - Asset Code
  - Vessel Name
  - Vessel Type
  - Classification
  - Capacity
  - Owner
  - Status (badge)
- **Pagination:** baris per page (default 10), prev/next, refresh button.

> **Catatan Phase 1:** Halaman ini **khusus Mother Vessel (MV)** — tidak ada tabs filter asset type lain (Barge, Tug Boat, dll). Seluruh list hanya menampilkan MV milik organisasi customer. Dukungan asset type lain dapat ditambahkan di phase berikutnya jika dibutuhkan (akan menjadi modul/scope terpisah).

### Status Badge (di tabel)

| Status | Badge variant | Keterangan |
|--------|---------------|------------|
| `ACTIVE` | Confirmed (hijau) | MV aktif, dapat dipakai di nominasi |
| `INACTIVE` | Neutral (abu-abu) | MV nonaktif (sudah di-deactivate, approved) |
| `PENDING APPROVAL` | Pending verify (kuning) | Submission baru, menunggu STS Admin |
| `REJECTED` | Error (merah) | STS Admin menolak; hover untuk lihat alasan |

### Form Add / Edit MV (Field Reference)

| Field | Required | Tipe | Catatan |
|-------|----------|------|---------|
| Asset Code | — | string | Auto-generate saat save (format `MV-XXXX`) |
| Vessel Name | ✅ | string | Free text |
| IMO Number | ✅ | string | 7-digit IMO |
| Type | ✅ | enum dropdown | e.g. Bulk Carrier, Cement Carrier, Tanker |
| Call Sign | ✅ | string | Free text |
| Classification | — | enum dropdown | BKI, GL, BV, dll. |
| Capacity (DWT) | ✅ | number | Dead Weight Tonnage |
| GRT | ✅ | number | Gross Register Tonnage |
| Length LOA (m) | ✅ | number | Length Over All |
| Beam (m) | ✅ | number | Vessel beam |
| Max Draft (m) | ✅ | number | Maximum draft |
| Owner | — | string | Auto-set ke organisasi customer, read-only |
| Status | ✅ | enum (Active/Inactive) | Default `Active` saat Add |

### Pending Approval Behavior

- Record `PENDING APPROVAL` **tetap tampil** di tabel Mother Vessel dengan badge khusus.
- Record `PENDING APPROVAL` **tidak bisa dipilih** sebagai MV di form Nomination Request (M8) — endpoint LPS hanya mereturn MV ber-status `ACTIVE` ke form M8.
- Tombol **Edit** dan **Deactivate** untuk record `PENDING APPROVAL` **disabled** (lock) sampai STS Admin approve/reject. Tooltip: "Edit tidak tersedia — sedang menunggu approval admin."
- Submit form Add baru tidak terblokir oleh pending request lain (customer bisa submit beberapa MV baru sekaligus).

### Audit Log per MV (View Detail)

Klik icon mata di tabel → modal/page detail MV. Section **"Riwayat Perubahan"** menampilkan daftar approval request untuk MV tersebut:

| Kolom | Deskripsi |
|-------|-----------|
| Timestamp Submit | ISO datetime aksi dibuat |
| Jenis Aksi | Add / Edit / Deactivate |
| Submitter | Nama user customer yang submit |
| Status | Pending / Approved / Rejected |
| Reviewer | Nama STS Admin yang verifikasi (kosong saat Pending) |
| Timestamp Resolution | ISO datetime approve/reject (kosong saat Pending) |
| Alasan Reject | Hanya saat status Rejected |

Data di-fetch dari STS API (endpoint approval request history per asset).

### Disclaimer di Form

Setiap form Add / Edit / Deactivate harus menampilkan disclaimer banner di top form:

> ⚠️ "All changes will be submitted as an approval request and require admin review before being applied."

## Batasan Modul

- **Tidak ada delete fisik:** Operasi delete tidak disediakan. "Deactivate" hanya mengubah `status` menjadi `INACTIVE`. Record tetap di STS untuk audit.
- **Tidak ada offline mode:** Aksi Add/Edit/Deactivate membutuhkan koneksi ke STS API. Jika STS unreachable, tampilkan error dan retry.
- **Tidak ada bulk operation:** Customer harus submit satu MV per aksi (tidak ada Add Multiple atau Bulk Deactivate di Phase 1).
- **Field yang bisa diedit terbatas:** Beberapa field master (e.g. IMO Number) mungkin di-restricted oleh STS Admin policy. Validasi sesuai aturan STS.
- **Khusus Mother Vessel (Phase 1):** Modul ini hanya mengelola asset type **MV**. Tidak ada tabs/filter/list untuk asset type lain (Barge, Tug Boat, Floating Crane, Bulldozer, Wheel Loader, Excavator). Dukungan type lain di luar scope Phase 1.
- **Owner field:** Auto-set ke organisasi customer yang login (dari JWT customer's organization_id). Customer tidak dapat mengubah owner.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-MV-01 | Sistem harus menyediakan menu **"Mother Vessel"** di sidebar Customer Portal yang menampilkan daftar Mother Vessel (MV) milik organisasi customer (data master di-fetch dari STS Platform via API) | Customer |
| FR-MV-02 | Halaman Mother Vessel harus menampilkan **list MV milik organisasi customer** dengan search (asset code, vessel name, IMO, owner) dan pagination. Phase 1 **khusus MV** — tidak ada tabs filter asset type lain | Customer |
| FR-MV-03 | Customer harus dapat menambahkan MV baru melalui tombol **"Add New MV"** yang membuka form dengan field: Asset Code (auto-generate saat save), Vessel Name, IMO Number, Type, Call Sign, Classification, Capacity (DWT), GRT, Length LOA (m), Beam (m), Max Draft (m), Owner (auto-set ke organisasi customer, read-only), Status. Submit form akan membuat **approval request** ke STS Platform via API | Customer |
| FR-MV-04 | Customer harus dapat mengedit MV existing miliknya melalui aksi Edit (icon pencil) yang membuka form yang sama dengan data ter-prefill. Submit edit akan membuat **approval request** ke STS Platform via API | Customer |
| FR-MV-05 | Customer harus dapat me-non-aktifkan (Deactivate) MV miliknya melalui aksi Deactivate (icon power). Aksi ini akan membuat **approval request** ke STS Platform untuk mengubah `status` MV menjadi `INACTIVE` | Customer |
| FR-MV-06 | Sistem harus menampilkan **disclaimer** di setiap form Add/Edit/Deactivate: "All changes will be submitted as an approval request and require admin review before being applied." | Customer |
| FR-MV-07 | Sistem harus mengirim seluruh aksi customer (Add / Edit / Deactivate MV) sebagai **approval request** ke STS Platform via API. LPS tidak menyimpan tabel master MV — data otoritatif tetap di STS. LPS hanya menyimpan cache status approval request dengan TTL singkat (default 5–15 menit) | System |
| FR-MV-08 | Sistem harus menampilkan **status approval** per MV di tabel list dengan badge yang sesuai: `ACTIVE` (hijau), `INACTIVE` (abu-abu), `PENDING APPROVAL` (kuning), `REJECTED` (merah). Status diambil dari STS API | Customer |
| FR-MV-09 | Record MV dengan status `PENDING APPROVAL` **tetap tampil** di tabel Mother Vessel, namun **tidak boleh dipilih** sebagai MV di form Nomination Request (M8). Validasi terjadi di sisi LPS (form M8 hanya menampilkan MV ber-status `ACTIVE`) | System |
| FR-MV-10 | Untuk MV ber-status `PENDING APPROVAL`, tombol Edit dan Deactivate harus **disabled** (lock) sampai STS Admin selesai verifikasi (approve atau reject). Hal ini mencegah race condition multiple approval request untuk record yang sama | System |
| FR-MV-11 | Sistem harus menyediakan view detail MV (icon mata) yang menampilkan section **"Riwayat Perubahan"** berisi daftar approval request untuk MV tersebut: timestamp submit, jenis aksi (Add/Edit/Deactivate), user yang submit, status (Pending/Approved/Rejected), alasan reject (jika ada), timestamp resolution. Data di-fetch dari STS API | Customer |
| FR-MV-12 | Sistem harus menampilkan **alasan reject** dari STS Admin pada MV ber-status `REJECTED` (hover state pada badge atau di section Riwayat Perubahan), agar customer mengetahui apa yang perlu diperbaiki sebelum re-submit | Customer |

---

## Role & Access

| Role | Akses |
|------|-------|
| Admin Kepelabuhan | View (tidak mengelola — STS Admin yang verifikasi) |
| Customer | Full (Add / Edit / Deactivate via approval request, hanya untuk MV miliknya) |
| SuperAdmin | View |
| Role lainnya | — |

---

## Dependency

- **M7 (Customer Authentication):** Customer harus login + JWT dengan organization_id. Owner field auto-set dari JWT.
- **M8 (Nomination Submission):** Form M8 mereferensikan MV dari modul ini (dropdown vessel). Validasi: hanya MV `ACTIVE` yang ditampilkan.
- **STS Platform API:**
  - `GET /api/sts/assets?owner_org_id=...&type=MV` — list MV milik customer (Phase 1 selalu `type=MV`).
  - `POST /api/sts/assets/approval-request` — submit Add/Edit/Deactivate approval request.
  - `GET /api/sts/assets/:id/approval-history` — riwayat approval request per MV.
  - `GET /api/sts/assets/:id` — detail MV.

---

## Asumsi & Open Questions

1. **Auto-generate Asset Code:** Sumber kebenaran format `MV-XXXX` ada di STS atau LPS? **Asumsi:** STS yang generate saat approve, customer melihat code final setelah `APPROVED`.
2. **Notifikasi approval result:** Bagaimana customer tahu approval-nya selesai? **Asumsi:** Toast/notification di sidebar saat customer login, plus polling cache TTL. Detail di spec.
3. **Edit MV `INACTIVE`:** Apakah customer bisa reactivate MV `INACTIVE`? **Asumsi:** Ya, via Edit form dengan ubah Status ke Active → approval request.
4. **Limit submission:** Apakah ada rate-limit jumlah approval request per customer per hari? **Asumsi:** Ditentukan STS, LPS forward error code.

---

## Referensi

- BRD Utama: [Section 2.1 (13)](../BRD-LPS-V3-Analysis.md) & [Section 3.4.13](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/mother-vessel-master/`](../../module/mother-vessel-master/) (tersedia)
- Implementation design: [`implementation/design/m13-mother-vessel-master-ui.md`](../../implementation/design/m13-mother-vessel-master-ui.md) (tersedia)
- Replit handoff: [`implementation/replit-handoff/m13-mother-vessel-master.md`](../../implementation/replit-handoff/m13-mother-vessel-master.md) (tersedia v1.0, contract STS ⚠ TENTATIVE)
- Modul terkait:
  - [M7 — Customer Authentication](m7-customer-authentication.md) (auth + org)
  - [M8 — Nomination Submission](m8-nomination-submission.md) (consumer MV dropdown)
  - [M10 — Customer Dashboard](m10-customer-dashboard.md) (potensi widget)
