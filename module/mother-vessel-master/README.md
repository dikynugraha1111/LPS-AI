# M13 — Mother Vessel Master (Customer Portal)

> **Catatan v1.0 (Mei 2026):** Modul BARU hasil briefing customer. Memungkinkan customer mengelola data master Mother Vessel (MV) miliknya sendiri di Customer Portal dengan workflow approval ke STS Platform.

## Overview

Modul dalam LPS Customer Portal yang menyediakan UI bagi customer untuk mengelola data master Mother Vessel (MV) miliknya: **Add MV**, **Edit MV**, dan **Deactivate MV**. Setiap aksi customer menghasilkan **approval request** yang dikirim ke STS Platform via API dan harus di-verifikasi oleh STS Admin sebelum berlaku efektif.

**Prinsip data ownership:** Master MV adalah single source of truth di STS Platform. **LPS tidak menyimpan tabel `mother_vessels`** di DB lokalnya — seluruh operasi list, detail, dan CRUD dilakukan via proxy ke STS API. LPS hanya menyimpan cache status approval request dengan TTL singkat (5–15 menit) untuk performa UI.

Phase 1 scope: **MV only**. Halaman ini khusus mengelola Mother Vessel — **tidak ada tabs/filter/list untuk asset type lain** (Barge, Tug Boat, Floating Crane, Bulldozer, Wheel Loader, Excavator). Dukungan asset type lain berada di luar scope Phase 1.

## Boundaries

**In scope:**
- Menu "Mother Vessel" di sidebar Customer Portal.
- List view MV milik organisasi customer + search (asset code, vessel name, IMO, owner) + pagination. **Tanpa tabs/filter asset type lain** — halaman khusus MV.
- **Add New MV:** form dengan field MV (Asset Code auto-generate, Vessel Name, IMO, Type, Call Sign, Classification, Capacity DWT, GRT, LOA, Beam, Max Draft, Owner auto-set, Status). Submit → approval request ke STS.
- **Edit MV:** form prefilled. Submit → approval request ke STS.
- **Deactivate MV:** konfirmasi modal. Submit → approval request ke STS (status `INACTIVE`).
- **View Detail MV:** modal/page dengan info MV + section "Riwayat Perubahan" (audit log approval request).
- Status badge: `ACTIVE` / `INACTIVE` / `PENDING APPROVAL` / `REJECTED`.
- Lock Edit/Deactivate saat record berstatus `PENDING APPROVAL` (mencegah race condition).
- Disclaimer banner di setiap form Add/Edit/Deactivate.
- Filter validasi di M8: hanya MV `ACTIVE` yang ditampilkan di vessel dropdown nomination.

**Out of scope:**
- Penyimpanan tabel master MV di DB LPS (master tetap di STS).
- Approval workflow internal di STS (proses verifikasi admin — milik STS Platform).
- Verifikasi field MV (e.g. IMO validation, capacity validation) — milik STS Admin policy.
- Asset type non-MV sepenuhnya (Barge, Tug Boat, Floating Crane, Bulldozer, Wheel Loader, Excavator) — tidak ada list/filter/CRUD; di luar scope Phase 1.
- Bulk operation (Add Multiple / Bulk Deactivate).
- Delete fisik record MV (hanya deactivate via status change).
- Notification engine (toast / push) — bagian dari shared notification infra LPS.

## Dependencies

| Dependency | Reason |
|-----------|--------|
| M7 Customer Authentication | Customer harus authenticated; `organization_id` dari JWT digunakan sebagai Owner auto-set |
| STS Platform API (outbound) | List MV, Add/Edit/Deactivate approval request, fetch approval history, detail MV |
| STS Platform Approval Queue | STS Admin meng-approve/reject request (proses ini di luar LPS) |
| System Config (M12) | `STS_ASSET_API_KEY`, `STS_APPROVAL_CACHE_TTL_SECONDS` (default 600) |

## Downstream

| Module | Dependency |
|--------|-----------|
| M8 Nomination Submission | Vessel dropdown form M8 hanya menampilkan MV milik customer ber-status `ACTIVE` |
| M10 Customer Dashboard | (Opsional) widget jumlah MV aktif / pending approval |

## Key Decisions

- **Data ownership: STS owns master, LPS proxies.** Konsisten dengan pola M9b Download EPB PDF (LPS tidak generate / menyimpan PDF) dan M12 master data sync. Mencegah duplikasi data & sinkronisasi divergen.
- **Phase 1 scope = MV only.** Halaman khusus Mother Vessel — **tidak ada tabs/filter/list untuk asset type lain** (Barge, Tug Boat, dll). Mempertahankan scope sempit & konsisten: kalau halaman menampilkan filter type lain, itu menyiratkan customer bisa berinteraksi dengannya padahal tidak. Dukungan type lain (jika dibutuhkan) menjadi modul/scope terpisah di kemudian hari.
- **Pending records tampil + lock + tidak bisa di-nominate.** Memberi transparency ke customer (lihat submission ada di queue) tapi mencegah penggunaan data yang belum verified di flow operasional (M8). Lock Edit/Deactivate saat pending mencegah race condition multiple approval request.
- **Audit log per MV (Riwayat Perubahan).** Section di detail MV menampilkan list approval request (timestamp, jenis aksi, submitter, status, reviewer, alasan reject). Critical untuk compliance & transparency. Data dari STS API.
- **Tidak ada delete fisik.** "Deactivate" hanya status → `INACTIVE`. Record tetap di STS untuk audit.
- **Owner auto-set, immutable.** Customer tidak bisa mengubah owner — auto dari JWT `organization_id`. Mencegah customer A mendaftarkan MV untuk customer B.
- **Disclaimer di setiap form.** Banner kuning di top form Add/Edit/Deactivate: "All changes will be submitted as an approval request and require admin review before being applied." Set ekspektasi customer bahwa aksi tidak instant.
- **Cache TTL singkat (5–15 menit).** LPS cache status approval request untuk performa UI; tidak menyimpan master MV. TTL pendek mencegah stale data setelah STS Admin verifikasi.

---

## UI/UX Design

**Surface A — Customer Portal** (Bahasa Indonesia untuk error/notif, English untuk field label sesuai screenshot referensi).

UI berada di menu **"Mother Vessel"** di sidebar Customer Portal (item baru). Mengikuti pattern Surface A: header page besar, search bar, data table, modal form. **Tanpa tabs filter asset type** — halaman khusus MV.

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m13-mother-vessel-master-ui.md`](../../implementation/design/m13-mother-vessel-master-ui.md) — akan dibuat

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen — terutama untuk form modal (field grid) dan status badge baru "Pending Approval".

## Status Implementasi

| Artefak | Status |
|---------|--------|
| BRD (utama + per-modul) | ✅ Done — [`document/brd/m13-mother-vessel-master.md`](../../document/brd/m13-mother-vessel-master.md) |
| Module folder (4 file) | ✅ Done (file ini, requirements.md, specifications.md, user-stories.md) |
| UI design doc | ✅ Done — [`implementation/design/m13-mother-vessel-master-ui.md`](../../implementation/design/m13-mother-vessel-master-ui.md) |
| Replit handoff | ✅ Done (v1.0, contract STS ⚠ TENTATIVE) — [`implementation/replit-handoff/m13-mother-vessel-master.md`](../../implementation/replit-handoff/m13-mother-vessel-master.md) |
| M8 update (filter vessel dropdown ke MV ACTIVE) | ✅ Done — spec + UI doc M8 diupdate (v1.1) + delta handoff [`m8-nomination-submission-v1.1-delta.md`](../../implementation/replit-handoff/m8-nomination-submission-v1.1-delta.md) |
