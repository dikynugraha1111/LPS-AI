# Replit Handoff — M8 Delta: Vessel Selector Terintegrasi M13

**Version:** 1.1
**Date:** 2026-05-16
**Parent handoff:** `m8-nomination-submission.md` (v1.0 → v1.1)

---

## What This File Is

File ini **hanya memuat perubahan** pada M8 yang dipicu oleh modul baru **M13 (Mother Vessel Master)**. Gunakan file ini jika M8 v1.0 sudah diimplementasi di Replit dan kamu hanya perlu menerapkan perubahan vessel selector.

Jika M8 belum diimplementasi sama sekali, kerjakan `m8-nomination-submission.md` (full handoff) lebih dulu, lalu terapkan delta ini.

**Dependency:** Modul **M13** harus tersedia — khususnya endpoint LPS internal `GET /api/customer/mother-vessels/active` (lihat `implementation/replit-handoff/m13-mother-vessel-master.md` Step 3.5). Jika M13 belum di-deploy, vessel selector tidak akan punya sumber data.

---

## Perubahan Inti

**Sebelum (v1.0):** Field "Vessel Name" = text input bebas (`vessel_name VARCHAR(255)`), atau combobox ke `GET /api/sts/vessels?search=` generik (semua vessel STS).

**Sesudah (v1.1):** Field "Mother Vessel" = **selector** dari **MV milik customer ber-status `ACTIVE`** via `GET /api/customer/mother-vessels/active` (endpoint M13). Customer tidak bisa ketik nama bebas; harus pilih dari MV yang sudah terdaftar & disetujui. MV `PENDING_APPROVAL`/`INACTIVE`/`REJECTED` tidak muncul (FR-MV-09).

**Rationale:** Master MV otoritatif di STS via M13. Nominasi harus mereferensikan MV yang valid & approved, bukan free-text yang bisa typo / tidak match master. Konsisten dengan single-source-of-truth.

---

## Prerequisites — Design Reference (WAJIB)

Baca sebelum ubah UI:
1. [`implementation/design/m8-nomination-submission-ui.md`](../design/m8-nomination-submission-ui.md) — **v1.1** (§4 Komponen "Vessel selector v1.1", §5 Edge Cases — empty state MV).
2. [`implementation/design/m13-mother-vessel-master-ui.md`](../design/m13-mother-vessel-master-ui.md) — konteks status MV (`ACTIVE`/`PENDING_APPROVAL`/dll).
3. [`module/nomination-submission/specifications.md`](../../module/nomination-submission/specifications.md) — §1.1 Vessel Selection.

---

## Step D1: Database Migration (delta)

File: `migrations/XXXXXX_m8_add_mv_asset_id.up.sql`

```sql
-- v1.1: nomination mereferensikan MV master STS (via M13). Tetap simpan snapshot untuk display & payload STS.
ALTER TABLE nominations ADD COLUMN IF NOT EXISTS mv_asset_id  VARCHAR(100);
ALTER TABLE nominations ADD COLUMN IF NOT EXISTS vessel_imo   VARCHAR(20);
-- vessel_name DIPERTAHANKAN (snapshot denormalized dari MV terpilih). Tidak di-drop (kompat draft lama).
```

> Tidak ada FK ke tabel MV — master MV TIDAK ada di DB LPS (milik STS via M13). `mv_asset_id` hanya referensi string ke asset STS. Draft legacy (sebelum v1.1) boleh `mv_asset_id` NULL — saat customer buka & re-submit, wajib pilih MV.

---

## Step D2: Backend — Validasi Submit (delta)

File: handler submit nominasi M8 (`internal/nomination/handler.go` atau setara).

Pada endpoint **Submit** (bukan Draft):
1. `mv_asset_id` **required** saat Submit (boleh NULL saat Draft).
2. Re-validate server-side: panggil internal service M13 (atau langsung adapter STS) untuk pastikan `mv_asset_id`:
   - milik customer (owner `organization_id` == JWT.org), DAN
   - ber-status `ACTIVE`.
3. Jika invalid (bukan milik customer / bukan ACTIVE / tidak ditemukan) → reject **422** `{error:"invalid_vessel", message:"MV tidak valid atau tidak aktif. Pilih MV aktif Anda."}`.
4. Saat valid: snapshot `vessel_name` + `vessel_imo` dari data MV ke row `nominations` (denormalized untuk payload STS & display historis — master tetap di STS).

> Reuse: panggil endpoint internal `GET /api/customer/mother-vessels/active` server-side, atau langsung `sts.AssetClient.GetAsset(mv_asset_id)` lalu cek owner+status. Pilih yang konsisten dengan arsitektur M13 (adapter layer `internal/sts/asset_client.go`).

**Draft:** `mv_asset_id` opsional. Jika diisi saat draft, tetap simpan (tanpa validasi ketat status — validasi penuh di Submit).

---

## Step D3: Frontend — Vessel Selector (delta)

File: form nominasi M8 (`src/pages/customer/nominations/NominationForm.tsx` atau setara), section "Informasi Kapal".

Ganti field "Vessel Name" lama:

- **Komponen:** Combobox shadcn (`Command` + `Popover`).
- **Data source:** `useQuery(['mv-active'], () => GET /api/customer/mother-vessels/active)`. Bukan `GET /api/sts/vessels`.
- **Option render:** `{vessel_name}` — baris kedua muted: `{asset_code} · IMO {imo_number}`.
- **On select:** simpan `mv_asset_id` ke form state. Auto-fill read-only display IMO (`vessel_imo`) + DWT (`capacity_dwt`) dari option terpilih.
- **Label:** "Mother Vessel" + asterisk required (bukan lagi "Nama Kapal" / "Vessel Name").
- **States:**
  - **Loading:** combobox disabled + spinner "Memuat daftar MV...".
  - **Empty (customer belum punya MV ACTIVE):** tampilkan di dalam popover/area: teks "Belum ada Mother Vessel aktif." + link button "Daftarkan MV di menu Mother Vessel →" → `navigate('/customer/mother-vessel')` (M13). Submit form di-block (validation: mv_asset_id required) selama belum ada MV terpilih.
  - **Error (fetch gagal — STS down via M13 proxy):** "Gagal memuat daftar Mother Vessel. Coba lagi." + retry button (refetch query).
- **Validation (zod / react-hook-form):** `mv_asset_id` required saat Submit; optional saat Save Draft. Pesan error: "Pilih Mother Vessel terlebih dahulu."

> Hapus referensi lama ke `GET /api/sts/vessels` di kode M8. Hapus free-text input `vessel_name` (diganti selector). `vessel_name` kini hanya nilai snapshot turunan dari pilihan MV (tidak di-input manual user).

---

## Step D4: Payload ke STS (delta)

Saat submit nominasi ke STS Platform: sertakan `mv_asset_id` (referensi master MV STS) di samping snapshot `vessel_name`/`vessel_imo`. STS bisa resolve ke master-nya via `mv_asset_id`. Field exact mengikuti contract STS nominasi existing (tidak berubah selain penambahan `mv_asset_id`).

---

## Acceptance Checklist (delta)

- [ ] Migration: `nominations` tambah `mv_asset_id`, `vessel_imo` (idempotent ADD COLUMN IF NOT EXISTS). `vessel_name` dipertahankan sebagai snapshot.
- [ ] Vessel field di form M8 = combobox dari `GET /api/customer/mother-vessels/active` (BUKAN `GET /api/sts/vessels`, BUKAN free-text).
- [ ] MV `PENDING_APPROVAL`/`INACTIVE`/`REJECTED` tidak muncul di selector (server-side filter di endpoint `/active`).
- [ ] Empty state: customer tanpa MV ACTIVE → pesan + link ke `/customer/mother-vessel`. Submit ter-block.
- [ ] Error state: fetch gagal → pesan + retry.
- [ ] On select: `mv_asset_id` tersimpan; IMO/DWT auto-fill read-only dari pilihan.
- [ ] Submit server-side: `mv_asset_id` required, re-validate owner==JWT.org + status ACTIVE, reject 422 jika invalid.
- [ ] Draft: `mv_asset_id` opsional (tidak block Save Draft).
- [ ] `vessel_name`/`vessel_imo` di-snapshot dari MV terpilih saat submit (untuk payload STS & display historis).
- [ ] Kode lama `GET /api/sts/vessels` & free-text vessel_name input dihapus dari M8.

---

## Catatan Dependency & TENTATIVE

- Bergantung pada M13 endpoint `GET /api/customer/mother-vessels/active` — pastikan M13 ter-deploy. Saat dev tanpa STS, M13 jalan dengan `STS_ASSET_MOCK=true` (lihat handoff M13) — selector M8 otomatis ikut pakai data mock.
- Contract STS untuk master MV masih ⚠ TENTATIVE (open questions di handoff M13). Perubahan M8 ini hanya konsumsi endpoint LPS internal yang stabil (`/api/customer/mother-vessels/active`) — tidak terdampak langsung oleh perubahan contract STS (terisolasi di adapter M13).
