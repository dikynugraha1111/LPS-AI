# M13 — Mother Vessel Master (Customer Portal): User Stories

> **v1.0 (Mei 2026):** Modul BARU. Phase 1 — MV only. User stories berfokus pada flow customer mengelola master MV dengan approval workflow ke STS.

## US-MV-01: View Mother Vessel List

**As a** Customer,
**I want to** see a list of all my organization's Mother Vessels with their current status,
**So that** I have visibility into the MV master data registered under my organization.

**Acceptance Criteria:**
- [ ] Sidebar Customer Portal menampilkan menu **"Mother Vessel"** (group "MASTER DATA"). Klik → navigate ke `/customer/mother-vessel`.
- [ ] Halaman menampilkan header "Mother Vessel" + subtitle "View and manage your organization's Mother Vessels. All changes require approval." + tombol primary "+ Add New MV" di kanan atas.
- [ ] Search bar dengan placeholder "Search by asset code, vessel name, IMO, or owner...".
- [ ] **Tidak ada tabs filter asset type dan tidak ada filter dropdown "All Assets"** — halaman khusus Mother Vessel; seluruh list hanya menampilkan MV milik organisasi customer.
- [ ] Tabel kolom: aksi (View / Edit / Deactivate icons) | Asset Code (mono) | Vessel Name | Vessel Type | Classification | Capacity | Owner | Status (badge).
- [ ] Pagination: "Baris: 10 ▾" + counter "Menampilkan 1–10 dari N data" + prev/next/page/refresh buttons.
- [ ] Search responsif (debounce 300ms).
- [ ] Empty state (belum ada MV sama sekali): "Belum ada Mother Vessel. Klik 'Add New MV' untuk menambahkan."
- [ ] Empty state (search tidak match): "Tidak ada MV yang cocok dengan pencarian Anda."
- [ ] Data di-fetch dari `GET /api/customer/mother-vessels?search={query}` (proxy ke STS, LPS hardcode `type=MV`).

## US-MV-02: Add New Mother Vessel

**As a** Customer,
**I want to** register a new Mother Vessel (MV) belonging to my organization,
**So that** I can use it in future nomination requests after admin approval.

**Acceptance Criteria:**
- [ ] Klik tombol "+ Add New MV" di header → modal **"Add New MV"** terbuka.
- [ ] Modal header: judul "Add New MV" + subtitle "Fill in the details below. This will create an approval request — changes will be applied after admin approval." + close icon (X).
- [ ] **Disclaimer banner** kuning di top form: "⚠ All changes will be submitted as an approval request and require admin review before being applied."
- [ ] Field form (2-column grid, mengikuti screenshot 2):
  - **Asset Code** (disabled, placeholder "MV-XXXX", helper text "Auto-generated on save")
  - **Vessel Name** (required, text input)
  - **IMO Number** (required, text input, validasi 7 digit numeric)
  - **Type** (required, dropdown, placeholder "Select type")
  - **Call Sign** (required, text input)
  - **Classification** (optional, dropdown, placeholder "Select classification")
  - **Capacity (DWT)** (required, numeric, placeholder "e.g. 50000")
  - **GRT** (required, numeric, placeholder "e.g. 30000")
  - **Length LOA (m)** (required, numeric, placeholder "e.g. 189")
  - **Beam (m)** (required, numeric, placeholder "e.g. 32")
  - **Max Draft (m)** (required, numeric, placeholder "e.g. 12.5")
  - **Owner** (disabled, prefilled dari organisasi customer, helper text "Automatically set to your organization")
  - **Status** (required, dropdown, default "Active")
- [ ] Tombol "Cancel" (outline) close modal tanpa save.
- [ ] Tombol "Submit for Approval" (primary navy) → POST ke `/api/customer/mother-vessels/approval-request` dengan `action: ADD`.
- [ ] Validasi inline per field: required, format IMO 7 digit, numeric > 0 untuk capacity/dimensi.
- [ ] Submit success → modal close, toast "MV submission terkirim. Menunggu approval admin." Tabel refresh, row baru muncul dengan badge **"Pending Approval"** kuning.
- [ ] Submit error 422 IMO duplicate → inline error di field IMO: "Nomor IMO sudah terdaftar."
- [ ] Submit error 5xx → toast error + retain form state untuk retry.

## US-MV-03: Edit Existing MV

**As a** Customer,
**I want to** edit details of an existing MV under my organization,
**So that** I can update outdated information (e.g. capacity, classification renewal).

**Acceptance Criteria:**
- [ ] Klik icon pencil (Edit) di row MV → modal **"Edit MV"** terbuka dengan field **prefilled** dari record current.
- [ ] Modal header: "Edit MV — {asset_code}".
- [ ] Disclaimer banner sama dengan US-MV-02.
- [ ] Asset Code field: **disabled** (tidak bisa diubah).
- [ ] Owner field: **disabled** (tidak bisa diubah).
- [ ] Field lain: editable, validasi sama dengan US-MV-02.
- [ ] Tombol "Submit for Approval" → POST `/api/customer/mother-vessels/approval-request` dengan `action: EDIT`, `asset_id`, full payload.
- [ ] Submit success → modal close, toast "Perubahan MV terkirim. Menunggu approval admin." Row di tabel update statusnya ke **"Pending Approval"** kuning.
- [ ] **Icon Edit di row dengan status `PENDING_APPROVAL`: disabled** + tooltip "Edit tidak tersedia — sedang menunggu approval admin." (Lihat US-MV-06.)
- [ ] Submit error 409 (sudah ada pending) → toast "Asset ini sedang menunggu approval. Edit akan tersedia setelah approval selesai."

## US-MV-04: Deactivate MV

**As a** Customer,
**I want to** deactivate an MV I no longer use,
**So that** it doesn't appear in my nomination vessel dropdown anymore.

**Acceptance Criteria:**
- [ ] Klik icon power (Deactivate) di row MV → modal konfirmasi sederhana: "Nonaktifkan MV {vessel_name}? Status akan diubah menjadi INACTIVE setelah approval admin."
- [ ] Disclaimer banner kuning di modal konfirmasi.
- [ ] Tombol "Cancel" + "Submit for Approval" → POST `/api/customer/mother-vessels/approval-request` dengan `action: DEACTIVATE`, `asset_id`.
- [ ] Submit success → modal close, toast "Permintaan deactivate terkirim. Menunggu approval admin." Row badge berubah ke **"Pending Approval"**.
- [ ] Untuk MV ber-status `INACTIVE`: icon power **berbeda warna/state** (atau icon "Reactivate") → klik akan submit `action: EDIT` dengan `status: ACTIVE` (lewat Edit form).
- [ ] Icon power di row dengan status `PENDING_APPROVAL`: **disabled** + tooltip sama dengan US-MV-03.

## US-MV-05: View MV Detail & Approval History

**As a** Customer,
**I want to** view the full detail of an MV and its approval history,
**So that** I can verify the registered information and audit past submission history (including rejection reasons).

**Acceptance Criteria:**
- [ ] Klik icon mata (View) di row → modal/page detail MV.
- [ ] Detail menampilkan semua field MV (read-only).
- [ ] Section **"Riwayat Perubahan"** dengan timeline atau table:
  - Per row: timestamp submit, jenis aksi (Add/Edit/Deactivate), submitter (nama user customer), status badge (Pending/Approved/Rejected), reviewer (STS Admin name), timestamp resolution, **alasan reject** (jika status Rejected).
  - Sorted descending (terbaru di atas).
- [ ] Data history di-fetch dari `GET /api/customer/mother-vessels/:id/approval-history`.
- [ ] Empty state history: "Belum ada riwayat perubahan untuk MV ini." (jarang, karena Add awal selalu jadi entry pertama).
- [ ] Saat status MV current = `REJECTED`: tampilkan banner merah di top detail dengan alasan reject + tombol "Edit & Resubmit" (open Edit modal dengan data terakhir).

## US-MV-06: Pending Approval Behavior

**As a** Customer,
**I want to** see that my submitted MV is awaiting admin approval and is locked from further changes,
**So that** I understand the queue status and avoid duplicate submissions.

**Acceptance Criteria:**
- [ ] Row MV ber-status `PENDING_APPROVAL` tampil di tabel Mother Vessel dengan badge **"Pending Approval"** (kuning soft pill).
- [ ] Icon Edit dan Deactivate di row tersebut: **disabled** (gray/opacity-50) + cursor `not-allowed`.
- [ ] Hover icon disabled → tooltip "Edit tidak tersedia — sedang menunggu approval admin."
- [ ] Klik icon View tetap **enabled** (customer boleh lihat detail + history).
- [ ] Row pending tetap tampil di semua tab filter yang relevan (e.g. tab "MV" tetap tampilkan pending MV).
- [ ] MV ber-status `PENDING_APPROVAL` **tidak muncul** di vessel dropdown form Nomination (M8). Validasi terjadi via endpoint `/api/customer/mother-vessels/active` yang filter `status=ACTIVE`.

## US-MV-07: View Approved Status & Use in Nomination

**As a** Customer,
**I want to** see that my MV submission has been approved and can now be used in nominations,
**So that** I know I can proceed with creating a nomination using this MV.

**Acceptance Criteria:**
- [ ] Setelah STS Admin approve: row MV di tabel berubah dari "Pending Approval" → **"Active"** (badge hijau).
- [ ] Update visible setelah cache expire (max 600s) atau manual refresh.
- [ ] (Opsional, future) In-app notification "MV {vessel_name} Anda telah disetujui dan siap digunakan untuk nominasi."
- [ ] MV `ACTIVE` muncul di vessel dropdown form M8 Nomination Request.
- [ ] Icon Edit dan Deactivate kembali **enabled** untuk row tersebut.

## US-MV-08: Handle Rejected Submission

**As a** Customer,
**I want to** see why my MV submission was rejected and have the option to fix and resubmit,
**So that** I can correct the issue and try again.

**Acceptance Criteria:**
- [ ] Setelah STS Admin reject: row MV di tabel berubah ke status **"Rejected"** (badge merah).
- [ ] Hover badge "Rejected" → tooltip menampilkan alasan reject (excerpt 100 char).
- [ ] Klik row atau icon View → buka detail; section Riwayat Perubahan menampilkan entry dengan status Rejected + alasan lengkap.
- [ ] Banner merah di top detail MV: "Submission ditolak: {alasan}. Klik 'Edit & Resubmit' untuk mengirim ulang."
- [ ] Tombol "Edit & Resubmit" (primary navy) → open Edit modal dengan data terakhir prefilled.
- [ ] Re-submit Edit → kembali ke flow approval (status "Pending Approval").
- [ ] Untuk record `REJECTED` yang merupakan **ADD** awal (asset belum pernah aktif): asset bisa di-delete soft (atau retain sebagai history). **Decision:** tetap retain di tabel dengan status Rejected agar customer bisa lihat history. Tidak muncul di nomination dropdown (filter `status=ACTIVE` only).

## US-MV-09: Search Mother Vessels

**As a** Customer,
**I want to** search my Mother Vessels by keyword,
**So that** I can quickly find a specific MV in a long list.

**Acceptance Criteria:**
- [ ] Search bar: ketik keyword → debounce 300ms → request ke API dengan `search={query}`. Match terhadap `asset_code`, `vessel_name`, `imo_number`, `owner_name`.
- [ ] Clear search button (X) di search bar saat field tidak kosong.
- [ ] Tidak ada tabs/filter asset type — halaman selalu menampilkan MV saja (search hanya mempersempit list MV).
- [ ] Search berlaku terhadap semua status MV (`ACTIVE`, `INACTIVE`, `PENDING_APPROVAL`, `REJECTED`).

## US-MV-10: M8 Nomination — MV Dropdown Filtered

**As a** Customer creating a Nomination Request,
**I want to** see only my Active MVs in the vessel dropdown,
**So that** I don't accidentally select an MV that's pending approval, inactive, or rejected.

**Acceptance Criteria:**
- [ ] Form M8 Nomination Request — vessel dropdown fetch dari `GET /api/customer/mother-vessels/active`.
- [ ] Dropdown hanya tampilkan MV ber-status `ACTIVE` milik customer.
- [ ] MV `PENDING_APPROVAL`, `INACTIVE`, `REJECTED`: **tidak muncul** di dropdown.
- [ ] Empty state dropdown (tidak ada MV active): "Belum ada MV aktif. Silakan daftarkan MV terlebih dahulu di menu Mother Vessel." + link ke `/customer/mother-vessel`.
- [ ] Server-side validation di M8 submit: re-check `vessel_id` valid + status `ACTIVE` sebelum create nomination. Reject 422 jika invalid (defensive — UI seharusnya prevent).

---

## Anti-stories (Out of Scope Phase 1)

- ❌ Bulk operation (Add Multiple MV sekaligus, Bulk Deactivate).
- ❌ Asset type non-MV sepenuhnya (Barge, Tug Boat, Floating Crane, Bulldozer, Wheel Loader, Excavator) — tidak ada list/filter/CRUD. Halaman khusus MV.
- ❌ Customer mengubah Owner field (security: auto-set dari JWT).
- ❌ Delete fisik record MV (hanya Deactivate via status change).
- ❌ Customer melihat MV milik organisasi lain (security: filter `owner_org_id` di endpoint).
- ❌ Approval workflow internal di LPS (STS Admin verifikasi di STS Platform, bukan LPS).
- ❌ Customer membatalkan pending approval request (defer ke phase 2; saat ini lock + tunggu admin).
