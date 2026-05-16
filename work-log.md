
## 2026-05-16T16:30:00+07:00
- **Task Performed:** Update M8 (Nomination Submission) — vessel selector beralih dari sumber generik (`GET /api/sts/vessels` / free-text) ke `GET /api/customer/mother-vessels/active` (MV milik customer ber-status `ACTIVE` via M13). Task terakhir pipeline M13. Scope: handoff delta terpisah (sesuai keputusan sebelumnya).
- **Files Modified:**
  - `module/nomination-submission/specifications.md` — tambah catatan v1.1. Field "Vessel Name (String)" → "Vessel (MV) Selection (mv_asset_id)". Tambah §1.1 "Vessel Selection (v1.1 — terintegrasi M13)": sumber endpoint, filter ACTIVE-only, snapshot vessel_name/imo, empty state, server-side validation submit, dependency M13. DB schema `nominations`: tambah kolom `mv_asset_id` + `vessel_imo` (vessel_name dipertahankan sebagai snapshot denormalized).
  - `implementation/design/m8-nomination-submission-ui.md` — bump last-updated + catatan v1.1. §4 Komponen Vessel selector di-rewrite (sumber `/api/customer/mother-vessels/active`, empty state link ke `/customer/mother-vessel`). Layout sketch "Nama Kapal [Cari kapal di STS]" → "Mother Vessel [Pilih MV aktif Anda]". §5 Edge Cases: pisah jadi 3 baris (MV list gagal dimuat, customer belum punya MV ACTIVE, anchor STS down).
  - `implementation/replit-handoff/m8-nomination-submission-v1.1-delta.md` — **FILE BARU** (pola M7 delta). 4 step delta: D1 migration (ADD mv_asset_id+vessel_imo, idempotent, vessel_name dipertahankan), D2 backend validasi submit (mv_asset_id required+owner+ACTIVE re-validate, 422 invalid, snapshot saat valid), D3 frontend combobox (data source, states loading/empty/error, validation), D4 payload STS (sertakan mv_asset_id). Acceptance checklist delta + catatan dependency M13/TENTATIVE.
  - `module/mother-vessel-master/README.md` — status implementasi: M8 update ✅ Done.
- **Logic / Decisions Made:**
  - **Delta handoff terpisah, bukan rewrite M8 full handoff.** Sesuai keputusan scope sebelumnya (M13 saja, M8 terpisah) + pola M7 delta yang sudah ada di repo. Mempermudah tracking perubahan & tidak menyentuh M8 v1.0 yang mungkin sudah diimplementasi.
  - **Source-of-truth docs (M8 spec + UI doc) diupdate in-place; handoff via delta.** Spec & UI doc adalah ground truth — harus selalu reflect state final (in-place edit dengan version note). Handoff adalah eksekusi incremental — delta lebih tepat. Konsisten dengan pipeline doctrine.
  - **`vessel_name` dipertahankan (tidak di-drop) sebagai snapshot denormalized.** Master MV milik STS; LPS simpan snapshot vessel_name+imo saat submit untuk display historis & payload STS tanpa harus re-fetch master tiap render. Draft legacy dengan vessel_name lama tetap kompat (mv_asset_id NULL diisi saat re-submit).
  - **Server-side re-validation di Submit (bukan hanya UI filter).** UI sudah filter ACTIVE-only, tapi endpoint submit M8 wajib re-validate owner==JWT.org + status ACTIVE (reject 422). Defensive — cegah manipulasi mv_asset_id via request langsung. FR-MV-09 di-enforce di dua lapis.
  - **mv_asset_id opsional saat Draft, required saat Submit.** Konsisten dengan M8 draft behavior existing (semua field longgar di draft, strict di submit).
  - **M8 hanya konsumsi endpoint LPS internal stabil** (`/api/customer/mother-vessels/active`) — tidak terdampak contract STS TENTATIVE (terisolasi di adapter M13). M8 delta bisa diimplementasi begitu M13 ter-deploy (mock atau real).
- **Results / Next Steps:**
  - **Pipeline M13 LENGKAP penuh:** BRD (utama + per-modul) ✅ · module 4-file ✅ · UI design ✅ (design system v1.4) · replit handoff M13 ✅ (v1.0 TENTATIVE) · integrasi M8 ✅ (spec+UI v1.1 + delta handoff). Semua artefak per pipeline doctrine `document/ → module/ → design/ → plan/ → replit-handoff/` tersedia.
  - **Blocker eksternal tersisa (bukan LPS-side):** 6 open questions API STS (`module/mother-vessel-master/specifications.md` §11 + handoff M13 akhir) — perlu konfirmasi tim STS sebelum contract final & production. Dev/test jalan dengan `STS_ASSET_MOCK=true`.
  - **Tidak ada next step LPS planning yang outstanding untuk M13.** Eksekusi Replit menunggu: (a) M13 di-deploy, (b) M8 delta diterapkan setelah M13 ada, (c) konfirmasi STS untuk lepas status TENTATIVE.

## 2026-05-16T14:00:00+07:00
- **Task Performed:** Buat replit handoff M13 (`implementation/replit-handoff/m13-mother-vessel-master.md` v1.0). Sebelumnya tanya user 3 hal: (1) penanganan 6 open questions API STS → asumsi + marker ⚠ TENTATIVE + adapter layer; (2) notif approval → webhook STS→LPS push (HMAC, pola M9b) dengan polling/cache fallback; (3) scope → M13 saja, M8 terpisah. Karena pilih webhook, spec M13 di-update dulu (polling-only → webhook utama) sebelum handoff (dual-update rule).
- **Files Modified:**
  - `module/mother-vessel-master/specifications.md` — bump note v1.0 → tambah catatan v1.1. §1 Approval Lifecycle diagram diupdate: tambah jalur webhook STS→LPS + sub-bagian "Mekanisme notifikasi" (webhook utama, polling fallback). Tambah §3.5 "STS Webhook — Inbound" (endpoint `POST /api/webhooks/sts/asset-approval-status`, HMAC, payload asumsi, processing idempotent + cache invalidation + notif). §4 endpoint table tambah baris webhook. §8 caching invalidation trigger tambah "webhook STS resolve" + catatan proaktif invalidation.
  - `implementation/replit-handoff/m13-mother-vessel-master.md` — **FILE BARU** v1.0. Header warning STS contract TENTATIVE + adapter layer mandate. 10 step: (1) migration cache-only (NO master MV table) + webhook idempotency log, (2) STS adapter layer isolasi asumsi + MockAssetClient flag, (3) customer endpoints proxy (list/detail/history/submit/active) dengan owner guard + validation + 409 lock, (4) webhook handler HMAC + idempotent + cache invalidation + notif, (5) sidebar nav, (6) list page, (7) Add/Edit modal, (8) Deactivate confirmation, (9) detail + Audit Timeline, (10) notifications. Acceptance checklist per area (data ownership, webhook, list, modals, a11y). Section "⚠ Open Questions Tim STS" (6 item) + instruksi `STS_ASSET_MOCK=true` untuk dev sebelum contract final.
  - `module/mother-vessel-master/README.md` — status implementasi: Replit handoff ✅ Done (TENTATIVE).
  - `document/brd/m13-mother-vessel-master.md` — status implementasi + referensi handoff updated.
- **Logic / Decisions Made:**
  - **Adapter layer (`internal/sts/asset_client.go`) mandatory** untuk isolasi contract STS yang masih asumsi. Saat STS confirm, hanya file ini berubah — handler & frontend tidak. `MockAssetClient` di belakang `STS_ASSET_MOCK=true` agar frontend bisa dikembangkan/tested tanpa block ke STS. Keputusan ini menjawab "tidak block progres" sambil jaga maintainability.
  - **Webhook STS→LPS dipilih user (bukan polling-only).** Konsekuensi: spec §1 lifecycle di-revise (dual-update rule wajib — tidak boleh handoff sebut webhook tapi spec masih polling). Pola HMAC + idempotency log reuse dari M9b (`verifySTSSignature`, `sts_asset_webhook_log` ON CONFLICT). Polling/cache TTL tetap ada sebagai fallback bila webhook drop (eventual consistency).
  - **Webhook handler boleh deploy duluan (idle).** STS mungkin belum siap kirim webhook saat go-live LPS — handler idle, sistem tetap jalan via fallback cache/polling. Mencegah hard dependency timing antar tim.
  - **Migration cache-only, eksplisit NO master MV table.** Ditegaskan di Step 1 + acceptance checklist item pertama — mencegah Replit Agent salah bikin tabel `mother_vessels` (anti-pattern: duplikasi master data milik STS).
  - **Scope M13 saja.** Endpoint `/active` tetap disiapkan di handoff ini (Step 3.5) tapi konsumsi oleh M8 ditunda ke handoff M8 terpisah — tidak nambah scope frontend M13.
  - **Open questions di-carry ke handoff, bukan di-resolve sepihak.** 6 pertanyaan STS dicantumkan eksplisit di akhir handoff dengan instruksi mock mode — transparan ke Replit Agent & tim, mencegah asumsi salah jadi production tanpa verifikasi.
- **Results / Next Steps:**
  - Pipeline M13 lengkap di sisi LPS planning: BRD ✅, module 4-file ✅, UI design ✅ (design system v1.4), replit handoff ✅ (v1.0 TENTATIVE).
  - **Blocker eksternal:** 6 open questions API STS (`specifications.md` §11 + handoff akhir) — perlu konfirmasi tim STS sebelum contract final & production. Sampai itu: dev/test pakai `STS_ASSET_MOCK=true`.
  - **Next step tersisa:** Update M8 (Nomination Submission) — vessel dropdown beralih ke `GET /api/customer/mother-vessels/active` (filter MV `ACTIVE`), empty state link ke `/customer/mother-vessel`. Update `module/nomination-submission/specifications.md` + `implementation/replit-handoff/m8-nomination-submission.md` (handoff delta terpisah, sesuai keputusan scope).

## 2026-05-16T10:00:00+07:00
- **Task Performed:** Buat UI design doc M13 (`implementation/design/m13-mother-vessel-master-ui.md`) + tambah 6 pattern reusable baru ke design system (v1.3 → v1.4). Mengikuti aturan CLAUDE.md UI/UX: baca design system master, invoke skill `ui-ux-pro-max` untuk validasi pattern form/dialog/disabled-state, pattern baru masuk design system dulu baru dipakai di per-modul doc.
- **Files Modified:**
  - `implementation/design/lps-design-system.md` — bump v1.3 → v1.4. §2.1 Status mapping tambah 4 baris: `ACTIVE`/`INACTIVE` (master data → Confirmed/Neutral), `PENDING_APPROVAL`/`REJECTED` (approval request → Pending verify/Error). §3.2 Molecules tambah 6 pattern baru: **Modal Dialog** (header/body-scroll/footer sticky, width tiers, focus trap, motion preset), **Confirmation Dialog** (icon bulat amber state-change vs rose destructive, copy konsekuensi konkret, default focus Batal), **Form Field Grid** (2-kolom, required asterisk, disabled vs read-only distinction, inline validation on-blur, field grouping >8 field), **Audit Timeline** (vertical timeline approval history, dot per status, rejection reason emphasized box), **Disclaimer Banner** (severity warning, pre-aksi expectation), **Locked Row Action** (disabled icon + tooltip + aria-label untuk pending-approval lock). Changelog v1.4 ditambah.
  - `implementation/design/m13-mother-vessel-master-ui.md` — **FILE BARU**. 11 section: ringkasan, page inventory (2 route + 4 modal), sidebar placement (grup MASTER DATA baru), halaman MV List (page header, search tanpa tabs, data table 8 kolom, action icons kontekstual per status, pagination, empty/error/loading states), modal Add/Edit MV (field reference 13 field di 3 grup, Add vs Edit behavior, validation on-blur, submit feedback, dismiss-with-unsaved-changes), modal Deactivate Confirmation, modal/page Detail + Riwayat Perubahan (Audit Timeline), component usage summary, edge cases, accessibility notes, cross-references.
  - `module/mother-vessel-master/README.md` — status implementasi: UI design doc ✅ Done.
  - `document/brd/m13-mother-vessel-master.md` — status implementasi + referensi UI design doc updated (tersedia).
- **Logic / Decisions Made:**
  - **6 pattern baru masuk design system, bukan langsung di per-modul doc.** Sesuai CLAUDE.md rule 6: pattern UI baru jadi reusable dulu. Modal Dialog, Confirmation Dialog, Form Field Grid, Audit Timeline, Disclaimer Banner, Locked Row Action — semua bisa dipakai modul lain (mis. M12 system config, future CRUD admin). Mencegah drift visual.
  - **Skill ui-ux-pro-max divalidasi:** design-system search konfirmasi arah "Data-Dense Dashboard / admin panel" (row hover, filtering, no ornate). UX domain search konfirmasi: inline validation **on blur** (bukan submit-only), disabled = `opacity + cursor-not-allowed`, confirmation wajib sebelum destruktif, submit feedback loading→toast, label wajib (no placeholder-only), field grouping logis. Semua diterjemahkan ke aturan konkret di design system v1.4.
  - **Deactivate = Confirmation Dialog amber, bukan rose destructive.** Deactivate bukan delete permanen (hanya status→Inactive setelah approval). Pakai icon amber + tombol primary navy, bukan rose/Trash. Membedakan "state change butuh approval" dari "delete permanen".
  - **Reactivate via Edit form, bukan endpoint terpisah.** MV `INACTIVE` → icon Deactivate berubah jadi Reactivate (emerald) → buka EditMVDialog prefilled Status=Active. Konsisten prinsip "everything is approval request" dari spec module.
  - **Edit enabled pada REJECTED (re-submit), Locked pada PENDING_APPROVAL.** REJECTED perlu jalur perbaikan (Edit & Ajukan Ulang). PENDING_APPROVAL dikunci untuk cegah race condition (FR-MV-10). View selalu enabled (read tidak pernah dikunci).
  - **Detail default modal, bukan halaman penuh.** Konsisten flow tabel (tidak kehilangan konteks list). Halaman penuh hanya untuk deep-link/refresh di route `/customer/mother-vessel/:id`.
  - **Field grouping 3 grup** (Identitas / Dimensi & Kapasitas / Kepemilikan) — form 13 field > 8 field, design system mewajibkan visual grouping via subheading.
- **Results / Next Steps:**
  - Design system v1.4 + UI design doc M13 sinkron. Pipeline M13: BRD ✅, module 4-file ✅, UI design ✅.
  - **Next steps tersisa:**
    1. `implementation/replit-handoff/m13-mother-vessel-master.md` — final executable prompt Replit Agent (full-stack: backend Go proxy ke STS API + frontend React modal/table/timeline). Wajib section "Prerequisites — Design Reference" rujuk design system v1.4 + UI doc M13.
    2. Update M8 (Nomination Submission) — vessel dropdown beralih ke `GET /api/customer/mother-vessels/active` (filter MV `ACTIVE` only). Update spec + handoff M8.
  - **Dependency eksternal belum resolved:** 6 open questions API STS di `module/mother-vessel-master/specifications.md` §11 — perlu konfirmasi tim STS sebelum replit handoff final (endpoint paths, asset_code generation, push vs polling, validasi field, rate limit, diff vs snapshot history).

## 2026-05-15T09:00:00+07:00
- **Task Performed:** Koreksi scope M13 berdasarkan feedback user — **hapus tabs filter asset type** dari halaman Mother Vessel. Phase 1 hanya MV, jadi menampilkan tabs/filter untuk type lain (Barge, Tug Boat, dll) tidak konsisten (menyiratkan customer bisa berinteraksi dengan type lain padahal tidak). Halaman jadi list MV murni dengan search saja. Rename sidebar menu "Asset Master" → "Mother Vessel" + route/API path + komponen agar mencerminkan scope. Konsistensi di-propagate ke seluruh pipeline.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — Section 2.1 (13) List View: hapus tabs filter + filter dropdown "All Assets"; "Scope Asset Type Phase 1" diganti "Scope Phase 1 — MV Only" (tegas: tidak ada tabs/filter/list type lain). Sidebar/header label "Asset Master" → "Mother Vessel". FR-MV-01/02 di-rewrite (menu "Mother Vessel", list + search, no tabs). FR-MV-09 "tabel Asset Master" → "tabel Mother Vessel". Change log #29 diupdate.
  - `document/brd/m13-mother-vessel-master.md` — List View section: hapus tabs filter + filter dropdown, tambah catatan Phase 1 (khusus MV). Sidebar/header "Mother Vessel". FR-MV-01/02 rewrite. FR-MV-09 + Pending Behavior "Asset Master" → "Mother Vessel". Batasan Modul: "Asset type non-MV view-only" → "Khusus Mother Vessel, tidak ada tabs/filter/list type lain". Status Implementasi + Referensi updated (module/ tersedia).
  - `module/mother-vessel-master/README.md` — overview, in-scope (menu "Mother Vessel", list tanpa tabs), out-of-scope (asset non-MV sepenuhnya, bukan "defer phase 2"), key decision Phase 1 (rationale: filter type lain menyiratkan interaksi), UI/UX section (tanpa tabs filter).
  - `module/mother-vessel-master/requirements.md` — scope note, FR-MV-01/02 rewrite, FR-MV-09 label fix.
  - `module/mother-vessel-master/specifications.md` — STS list API `type` jadi required `MV` (Phase 1 note). LPS endpoint `/api/customer/assets/*` → `/api/customer/mother-vessels/*` (active-mv → active). Query param `type` dihapus dari list endpoint (hardcode MV). Frontend route `/customer/assets` → `/customer/mother-vessel`, komponen `AssetMasterPage` → `MotherVesselPage`. Sidebar menu label + icon. Cache key list buang `type`.
  - `module/mother-vessel-master/user-stories.md` — US-MV-01 "View Asset Master List" → "View Mother Vessel List" (no tabs, no filter dropdown, empty state disesuaikan). US-MV-06/11 label fix. US-MV-09 "Filter & Search Assets" → "Search Mother Vessels" (search only, no tabs). **US-MV-10 "View-Only for Non-MV Asset Types" DIHAPUS** (tidak relevan — tidak ada non-MV). US-MV-11 → US-MV-10 (renumber). API paths + label propagate. Anti-story non-MV dipertegas.
- **Logic / Decisions Made:**
  - **Hapus tabs filter sepenuhnya, bukan disable.** User benar: menampilkan tabs type lain di halaman yang hanya support MV adalah misleading UX. Halaman jadi single-purpose "Mother Vessel" list. Lebih jujur ke customer & lebih simpel di-implement.
  - **Rename menu/route/API "Asset Master" → "Mother Vessel".** Nama generik "Asset Master" mengimplikasikan multi-asset. Karena scope MV-only, semua naming (sidebar menu, route `/customer/mother-vessel`, API `/api/customer/mother-vessels`, komponen `MotherVesselPage`) diselaraskan. Nama modul/folder resmi tetap "Mother Vessel Master" / `mother-vessel-master` (sudah deskriptif).
  - **STS list API `type=MV` jadi required + hardcoded LPS-side.** LPS tidak pernah call STS untuk type lain. STS API boleh tetap support multi-type untuk konsumen lain — itu di luar concern LPS Phase 1.
  - **US-MV-10 dihapus, bukan di-repurpose.** Story "lihat asset non-MV view-only" tidak ada artinya jika halaman tidak menampilkan non-MV sama sekali. Renumber US-MV-11 → US-MV-10 supaya rapat.
  - **Dual-update rule dipatuhi:** BRD utama + BRD per-modul diupdate di sesi yang sama.
- **Results / Next Steps:**
  - Seluruh pipeline M13 (BRD utama, BRD per-modul, 4 module file) konsisten: halaman khusus Mother Vessel, tanpa tabs/filter asset type lain.
  - Final sweep grep memverifikasi tidak ada lagi referensi tabs/filter aktif (sisa match hanya kalimat negasi "tidak ada tabs filter").
  - Next steps tidak berubah: (1) UI design doc M13, (2) replit handoff M13, (3) update M8 vessel dropdown ke `/api/customer/mother-vessels/active`.

## 2026-05-14T23:15:00+07:00
- **Task Performed:** Buat module folder `module/mother-vessel-master/` dengan 4 file (4-file rule) untuk modul M13. File mencakup README (boundaries + dependencies + key decisions), requirements (FR table + cross-cutting NFR), specifications (lifecycle + STS API contract + LPS endpoints + caching + error handling + open questions), dan user-stories (11 US + anti-stories).
- **Files Modified:**
  - `module/mother-vessel-master/README.md` — **FILE BARU**. Overview, boundaries (in/out scope), dependencies (M7, STS API, M12 config), downstream (M8 vessel dropdown, M10 widget), key decisions (data ownership STS-side, phase 1 MV only, pending behavior, audit log, no physical delete, owner immutable, disclaimer, cache TTL), UI design reference, status implementasi.
  - `module/mother-vessel-master/requirements.md` — **FILE BARU**. Tabel FR-MV-01..FR-MV-12 dengan kolom Priority "Must Have". Cross-cutting requirements (STS integration latency, real-time, auditability, security owner validation, fallback STS unreachable, data ownership prinsip).
  - `module/mother-vessel-master/specifications.md` — **FILE BARU**. 11 section: lifecycle ASCII diagram, data ownership (no master MV table di LPS, hanya cache TTL 600s), STS API contract (4 endpoint dengan asumsi tentative — list, detail, submit approval, history), LPS customer-facing API (5 endpoint termasuk `/active-mv` khusus M8), frontend routes + modal/dialog inventory, sidebar menu spec, **M8 integration spec** (vessel dropdown filter ke `active-mv`), caching strategy table, error handling table, security (JWT validation, STS_API_KEY server-side only, rate limiting), open questions untuk tim STS.
  - `module/mother-vessel-master/user-stories.md` — **FILE BARU**. 11 user stories: US-MV-01 view list (sidebar + tabs + search + table), US-MV-02 Add New MV (modal form lengkap dengan 13 field + disclaimer), US-MV-03 Edit MV (prefilled, lock saat pending), US-MV-04 Deactivate (konfirmasi modal), US-MV-05 view detail + Riwayat Perubahan, US-MV-06 pending behavior (lock + tidak muncul di M8 dropdown), US-MV-07 approved status, US-MV-08 rejected + resubmit flow, US-MV-09 filter & search, US-MV-10 non-MV view-only (Phase 1), US-MV-11 M8 nomination dropdown filtered. Plus anti-stories.
- **Logic / Decisions Made:**
  - **Tetap di folder kebab-case "mother-vessel-master"** (bukan "asset-master") agar nama folder mencerminkan scope Phase 1 (MV-focused), tapi sidebar menu pakai label "Asset Master" sesuai screenshot referensi yang lebih general (untuk akomodasi expansion ke type lain di phase berikutnya tanpa rename folder).
  - **API contract di specifications.md ditandai "tentative" (asumsi)** karena tim STS belum konfirmasi endpoint final. Open Questions section di akhir spec mencatat 6 hal yang perlu klarifikasi tim STS sebelum implementasi: endpoint paths, asset_code generation, push vs polling, validasi field STS, rate limit, diff vs snapshot di history.
  - **Endpoint terpisah `/api/customer/assets/active-mv` untuk M8 integration** — bukan reuse general list endpoint. Alasan: (1) payload lebih ringan (hanya field essential untuk dropdown), (2) filter eksplisit `status=ACTIVE` di server-side mencegah leak status lain ke nomination flow, (3) cache TTL berbeda dari general list (lebih agresif refresh 60s vs 300s).
  - **Phase 1 view-only untuk asset type non-MV** dilengkapi tooltip "Pengelolaan {type} akan tersedia di phase berikutnya." Jelas mengatakan scope expansion ke customer.
  - **Anti-stories ditulis eksplisit** di user-stories.md untuk lock scope: no bulk operation, no non-MV CRUD, no owner change, no physical delete, no cross-org access, no LPS-side approval workflow, no cancel pending request.
  - **Reactivate MV `INACTIVE`** lewat Edit form (`status: ACTIVE`) — tidak ada endpoint khusus "Reactivate". Konsisten dengan principle "everything is approval request".
  - **Cache opsional** di specifications. Replit Agent boleh implement tanpa Redis kalau STS API < 200ms — fallback ke direct proxy. Tidak block delivery Phase 1.
- **Results / Next Steps:**
  - 4-file rule terpenuhi: README.md (6.3K), requirements.md (5.3K), specifications.md (14.9K), user-stories.md (13.0K).
  - Total folder size compact, semua artefak BRD-level + module-level untuk M13 sudah lengkap.
  - **Next steps yang masih pending:**
    1. `implementation/design/m13-mother-vessel-master-ui.md` — UI design doc (Surface A) dengan layout list page, modal form Add/Edit, modal Deactivate konfirmasi, detail page + Riwayat Perubahan section, status badge mapping baru untuk PENDING_APPROVAL & REJECTED.
    2. (Opsional) Update `implementation/design/lps-design-system.md` jika perlu pattern baru "Approval Request Status Badge" atau "Disclaimer Banner".
    3. `implementation/replit-handoff/m13-mother-vessel-master.md` — final executable prompt untuk Replit Agent (full-stack: API proxy backend Go + frontend React modal/page).
    4. Update M8 spec + handoff — vessel dropdown beralih ke `GET /api/customer/assets/active-mv`. Update form M8 untuk handle empty state (link ke `/customer/assets` jika tidak ada MV active).
  - **Dependency eksternal yang perlu ditindaklanjuti sebelum Replit handoff:** Konfirmasi tim STS untuk 6 open questions di `module/mother-vessel-master/specifications.md` §11.

## 2026-05-14T22:30:00+07:00
- **Task Performed:** Tambah **modul baru M13 — Mother Vessel Master (Customer Portal)** di BRD. Modul memungkinkan customer mengelola data master MV miliknya (Add/Edit/Deactivate) dengan workflow approval ke STS Platform. Data master MV tetap di DB STS — LPS hanya proxy. Phase 1 scope: MV only. Briefing dari user dengan 3 screenshot referensi (Asset Master list view, Add New MV form, flow diagram approval).
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — tambah change log entry #29. Section 1.3 (LPS-STS table) tambah baris "Mother Vessel Master (Customer)". Section 2.1 tambah sub-section "13. Mother Vessel Master (Customer Portal)" lengkap (definisi, tujuan, cakupan, lifecycle approval, list view spec, status badge, pending behavior, audit log, catatan integrasi). Section 3.1 update count: "12 modul" → "14 modul" + tabel modul tambah baris M13. Section 3.3.2 Module Access Matrix tambah baris Mother Vessel Master. Section 3.4 tambah sub-section "3.4.13" dengan tabel FR-MV-01..FR-MV-12.
  - `document/brd/m13-mother-vessel-master.md` — **FILE BARU**. BRD per-modul mengikuti pola m9b/m9c: definisi, tujuan, cakupan (3 customer capabilities + lifecycle diagram + list view spec + status badge + form field table + pending behavior + audit log section + disclaimer), batasan modul, tabel FR-MV-01..FR-MV-12, role & access, dependency (M7, M8, STS API endpoints), asumsi & open questions, referensi.
  - `CLAUDE.md` — tabel "BRD per modul" tambah baris M13.
- **Logic / Decisions Made:**
  - **Data ownership: STS owns, LPS proxies.** Konsisten dengan pola M9b (Download EPB PDF) dan M12 (master data sync). LPS tidak menyimpan tabel `mother_vessels` — cukup cache status approval request (TTL 5–15 menit). Otoritatif di STS. Mencegah duplikasi data & sinkronisasi divergen.
  - **Phase 1: MV only.** Sesuai jawaban user. Tabs Barge/Tug Boat/Floating Crane/Bulldozer/Wheel Loader/Excavator tetap muncul untuk filter view (data dari STS master), tapi Add/Edit/Deactivate hanya MV. Mengurangi scope phase 1, form dynamic per-type bisa di-defer ke phase 2.
  - **Pending records tampil dengan badge "Pending Approval" + tidak bisa dipilih di nominasi.** Memberi transparency ke customer (lihat submission-nya ada di queue) tapi mencegah penggunaan data yang belum verified di flow operasional (M8 nomination).
  - **Lock saat PENDING.** Edit/Deactivate disabled saat record `PENDING APPROVAL`. Mencegah race condition multiple approval request untuk satu record. STS Admin tidak perlu adjudicate stack request.
  - **Audit log per MV (Riwayat Perubahan).** Section di detail MV menampilkan list approval request: timestamp submit, jenis aksi, submitter, status, reviewer, alasan reject. Critical untuk compliance & transparency customer. Data dari STS API (LPS tidak duplikat history).
  - **Tidak ada delete fisik.** "Deactivate" hanya status → INACTIVE. Record tetap untuk audit. Konsisten dengan prinsip immutability di M12 audit log.
  - **Owner field auto-set.** Customer tidak bisa pilih owner — auto dari JWT customer's organization_id. Mencegah customer A mendaftarkan MV untuk customer B.
  - **Disclaimer di setiap form.** Banner kuning di top form: "All changes will be submitted as an approval request and require admin review before being applied." Set expectation customer bahwa aksi tidak instant.
  - **STS API endpoint asumsi (untuk dokumentasi):** `GET /api/sts/assets`, `POST /api/sts/assets/approval-request`, `GET /api/sts/assets/:id/approval-history`, `GET /api/sts/assets/:id`. Detail endpoint final menunggu spec STS — dicatat sebagai open question.
- **Results / Next Steps:**
  - BRD utama v3.5 sekarang punya modul M13. BRD per modul (m13-mother-vessel-master.md) sudah dibuat. Mapping di CLAUDE.md updated.
  - **Next steps (belum dilakukan, menunggu approval user):**
    1. `module/mother-vessel-master/` — buat 4 file (README, requirements, specifications, user-stories) per 4-file rule.
    2. `implementation/design/m13-mother-vessel-master-ui.md` — UI design doc (Surface A, layout sesuai screenshot referensi).
    3. `implementation/design/lps-design-system.md` — mungkin tambah pattern "Approval Request Status Badge" jika belum ada.
    4. `implementation/replit-handoff/m13-mother-vessel-master.md` — final executable prompt untuk Replit Agent.
    5. Update M8 (Nomination Submission) — vessel dropdown harus filter MV ber-status `ACTIVE` only.
  - **Dependency eksternal:** Spec API STS untuk asset master + approval workflow. Tim STS perlu konfirmasi endpoint, payload, dan field validation rules sebelum implementasi.

## 2026-05-14T21:30:00+07:00
- **Task Performed:** Tambah section **"Detail Nominasi"** di halaman EPB detail (M9b) + pertegas spec Payment Instruction Box (Block 3) sesuai referensi screenshot production. BRD naik ke v3.5, M9b handoff naik ke v2.2.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — tambah change log entry #28. Tambah FR-EI-13 di section 3.4.9b.
  - `document/brd/m9b-epb-invoice.md` — bump v3.4 → v3.5. Tambah catatan v3.5 di top. Tambah FR-EI-13 di tabel Functional Requirements.
  - `module/epb-invoice/specifications.md` — tambah note v3.5 di top. Response `GET /api/customer/epb-payments/:id` diperluas dengan field `nomination_data` (vessel_name, vessel_type, cargo_type, towage_plan, eta, agent_name, charterer, created_at, updated_at). Keterangan JOIN 3-tabel di endpoint detail.
  - `implementation/design/m9b-epb-invoice-ui.md` — bump v3.4 → v3.5. Tambah note v3.5. Layout diagram di §5 diupdate: section "Detail Nominasi" ditambahkan di antara Status Banner dan Invoice Detail Card. Tambah §5.1 "Detail Nominasi Section" (grid 2-col, field mapping, styling, fallback). Renumber §5.2–§5.5. Tambah spec exact Block 2 tfoot (Total bold + navy, border-t-2) dan Block 3 Instruksi Pembayaran (2-column grid, layout field exactc). Komponen table tambah "Detail Nominasi grid". Edge cases tambah `nomination_data` null handling.
  - `implementation/replit-handoff/m9b-epb-invoice.md` — bump v2.1 → v2.2. Version history. Step 4 `GET /api/customer/epb-payments/:id` ditambah SQL 3-tabel JOIN + field nomination_data. Step 9 ditambah Section "Detail Nominasi" dengan JSX snippet (SectionCard + dl grid). Block 2 tfoot JSX dipertegas (Total border + color). Block 3 Payment Instruction Box JSX dipertegas (2-column grid, exact className per field). Acceptance checklist tambah section "v2.2 additions" (backend 3 item + frontend 7 item).
- **Logic / Decisions Made:**
  - **Nomination data dari JOIN, bukan duplikasi:** Data nominasi (vessel_name, vessel_type, cargo_type, dll.) sudah ada di tabel `nominations`. API endpoint detail cukup JOIN 3-tabel (`epb_payments` ⋈ `nomination_epb` ⋈ `nominations`) — tidak perlu kolom baru di `epb_payments`. Prinsip single source of truth terjaga.
  - **Section selalu tampil (non-conditional):** Berbeda dengan Invoice Detail Card blocks (yang kondisional berdasarkan data STS), "Detail Nominasi" selalu ada karena nomination data selalu tersedia. Exception: null safety jika legacy API belum return `nomination_data` — section disembunyikan, bukan crash.
  - **Block 3 Payment Instruction Box — 2-column layout:** Berdasarkan screenshot referensi user, layout dalam box amber adalah 2-column: kiri (Bank, No Rek, Atas Nama) vs kanan (Kode Bayar, Batas Pembayaran, Total). Total di-highlight bold + navy `text-[#0F2A4D]` untuk penekanan visual pembayaran. Kode Bayar + No Rek font-mono untuk mudah salin.
  - **Block 2 tfoot Total row:** Border `border-t-2 border-[#0F2A4D]` di atas baris Total untuk separasi visual kuat antara line items dan grand total. Sesuai referensi screenshot screenshot 3.
  - **formatDateTime helper:** Perlu utility baru `formatDateTime(iso: string): string` mengembalikan `dd MMM yyyy HH:mm` (format yang ditampilkan di screenshot production "09 Mei 2026 pukul 15.24 WIB"). Diletakkan di `src/lib/format.ts` bersama `formatCurrency` dan `dueDateCountdown`.
- **Results / Next Steps:**
  - BRD v3.5, specs, UI doc, dan handoff v2.2 sinkron full pipeline.
  - Replit Agent perlu implement: (1) tambah JOIN `nominations` di SQL query `GET /api/customer/epb-payments/:id`, (2) serialize `nomination_data` di response, (3) render Section "Detail Nominasi" di `EPBDetailPage.tsx` (atas Invoice Detail Card), (4) update tfoot Block 2 styling, (5) update Block 3 Payment Instruction Box ke 2-column grid layout.
  - **Dependency:** Tidak ada perubahan DB schema baru (semua field yang dipakai sudah ada di tabel `nominations`). Tidak perlu koordinasi STS. Safe to deploy standalone.

## 2026-05-14T20:00:00+07:00
- **Task Performed:** Upgrade Detail Tagihan EPB ke **invoice-style display** + tambah aksi Download EPB PDF + multi-currency support (IDR/USD). Berdasarkan permintaan user dengan referensi screenshot 2 (production invoice layout). Improvement berimpact ke M9 dan M9b. BRD naik ke v3.4, design system naik ke v1.3, M9 handoff naik ke v2.2, M9b handoff naik ke v2.1.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — bump v3.3 → v3.4. Tambah change log entry #27. Section 2.1 (9) EPB Detail diperluas dengan list invoice-style data (Vessel Ops, Line Items, kalkulasi, Instruksi Pembayaran, Download PDF). Section 2.1 (9b) tambah sub-section "Data EPB yang Ditampilkan (Invoice-style, v3.4)" + "Aksi Customer di Detail EPB" + "Currency Support (v3.4)". Tambah FR-NP-09 (preview EPB invoice-style di M9 + Download). Tambah FR-EI-11 (invoice-style Detail Tagihan di M9b) + FR-EI-12 (Download EPB PDF proxy).
  - `document/brd/m9b-epb-invoice.md` — bump v3.3 → v3.4. Tambah catatan v3.4 di top. Section Cakupan tambah "Data EPB yang Ditampilkan (Invoice-style, v3.4)" dengan tabel field + "Aksi Customer di Detail EPB" + "Currency Support". Tambah FR-EI-11 dan FR-EI-12 di tabel Functional Requirements.
  - `document/brd/m9-nomination-status.md` — bump v3.2 → v3.4. Tambah catatan v3.4. Section Cakupan EPB Detail diperluas dengan invoice-style data + Download button. Tambah FR-NP-09.
  - `module/epb-invoice/requirements.md` — tambah scope note v3.4. Tambah FR-EI-11 dan FR-EI-12 di tabel.
  - `module/epb-invoice/specifications.md` — tambah note v3.4 di top. Section 7 "List & Detail EPB": response detail diperluas dengan `vessel_ops`, `line_items[]`, `subtotal`, `vat_*`, `bank_info`, `epb_pdf_url`. Tambah endpoint baru `GET /api/customer/epb-payments/:id/document` (proxy stream PDF). Endpoints summary diupdate.
  - `module/nomination-status-payment/specifications.md` — tambah scope note v3.4. Webhook payload APPROVED diperluas (vessel_ops, line_items, subtotal, vat_*, total_amount, bank_info, epb_pdf_url) + catatan backwards compatibility. DB schema `nomination_epb` diperluas dengan kolom kalkulasi + voyage ops + bank info + pdf_url. Tabel baru `nomination_epb_line_items`.
  - `implementation/design/lps-design-system.md` — bump v1.2 → v1.3. Tambah 3 komponen baru di §3.2 setelah Voyage Card: **Invoice Detail Card (Surface A, v1.3)** dengan 3 blok (Vessel Ops Grid + Line Items Table + Payment Instruction Box) lengkap dengan JSX snippet + aturan pakai (currency formatter, compact variant, fallback minimal), **Payment Instruction Box (standalone)** dengan countdown indicator rules, **Line Items Table (standalone)**. Changelog v1.3 ditambah.
  - `implementation/design/m9b-epb-invoice-ui.md` — header bump v3.4. Section 5 Halaman EPB Detail di-rewrite: layout 2-column diganti dengan Invoice Detail Card menggantikan section Detail Tagihan + Bank Tujuan terpisah. Section 5.0 baru: "Header & Download Action" dengan tombol Download EPB PDF (selalu tampil, FR-EI-12). Block 3 Payment Instruction hanya tampil saat UNPAID/PAYMENT_REJECT. Component Usage Summary diupdate dengan 3 komponen baru. Edge Cases tambah: legacy fallback, pdf_url null, currency USD/IDR, batas pembayaran ≤ 3 hari.
  - `implementation/design/m9-nomination-status-payment-ui.md` — header bump v3.4. Section 3.4 EPB Detail Section di-rewrite: pakai Invoice Detail Card **compact variant** (Block 1 + 2, tanpa Block 3) dengan tombol Download EPB PDF di header + Bayar EPB di Action Card right column. Component Usage Summary diupdate.
  - `implementation/replit-handoff/m9b-epb-invoice.md` — bump v2.0 → v2.1. Version history. Step 9 EPB Detail Page di-rewrite penuh: payload response shape baru (vessel_ops, line_items, bank_info, epb_pdf_url), spec render Invoice Detail Card (3 blok kondisional), header dengan tombol Download EPB PDF. Helper `formatCurrency()` + `dueDateCountdown()` di `src/lib/format.ts`. Endpoint proxy backend baru `GET /api/customer/epb-payments/:id/document`. Acceptance checklist diperluas dengan 10+ item baru untuk invoice-style + Download endpoint + multi-currency.
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — bump v2.1 → v2.2. Step 1 migration di-rewrite: `nomination_epb` extended dengan 17 kolom baru + tabel baru `nomination_epb_line_items` + delta migrasi (ALTER TABLE ADD COLUMN IF NOT EXISTS untuk DB existing). Step 3 webhook handler APPROVED: payload Go struct extended (VesselOps, LineItem, BankInfo) + insert ke kedua tabel + backwards compatibility note. Step 4 response API include extended EPB fields + `epb_payment_id` + `payment_status` + `epb_pdf_url_available` flag. Step 7 frontend EPB Detail Section di-rewrite: pakai Invoice Detail Card compact + Download EPB PDF di header + Bayar EPB navigate ke `/customer/billing/epb/:epb_payment_id`. Acceptance checklist tambah section "v2.2 additions".
- **Logic / Decisions Made:**
  - **Data ownership separation:** Data invoice-style (vessel_ops, line_items, ppn, bank_info, pdf_url) **disimpan di `nomination_epb` (owned by M9)**, bukan di `epb_payments` (M9b). Alasan: data ini berasal dari STS webhook APPROVED yang sudah ditangani M9. M9b cukup JOIN saat detail dibutuhkan. Mencegah duplikasi data dan menjaga prinsip "STS owns billing data; LPS owns payment workflow".
  - **Compact variant di M9 vs Full di M9b:** Invoice Detail Card di M9 (preview di halaman nominasi) hanya tampilkan Block 1 + Block 2 (tanpa Payment Instruction Box). Instruksi pembayaran lengkap (bank, kode bayar, batas) hanya muncul di M9b setelah customer commit ke "Bayar EPB". Mencegah duplicate visual noise dan jelaskan separation antara "lihat status" vs "lakukan bayar".
  - **Block 3 Payment Instruction kondisional:** Hanya tampil saat status = UNPAID atau PAYMENT_REJECT. Saat WAITING_PAYMENT_VERIFICATION / PAID, customer tidak perlu transfer lagi, jadi block disembunyikan agar tidak menyesatkan. Status badge dan banner sudah cukup menjelaskan keadaan.
  - **Download EPB PDF = proxy STS, bukan generate LPS:** LPS tidak generate PDF mandiri (out-of-scope billing). Endpoint `/document` fetch dari STS dengan Bearer auth server-side, stream ke client. Mencegah expose STS URL & auth ke browser. Tombol disabled + tooltip saat `epb_pdf_url` null (STS belum upload dokumen).
  - **Currency support multi-currency (IDR/USD):** Field `currency` dari payload STS jadi sumber kebenaran. UI render via `Intl.NumberFormat` dengan locale berbeda (id-ID untuk IDR, en-US untuk USD). Validasi min partial payment tetap "1 USD ekuivalen": jika currency USD pakai 1 USD langsung, jika IDR pakai 1 × `USD_IDR_RATE` (atau fallback). Tidak ada konversi balik IDR→USD untuk display.
  - **Backwards compatibility wajib:** Field invoice-style dari STS bersifat optional di payload. Jika STS belum upgrade (legacy nomination), kolom DB NULL dan UI fallback ke pattern minimal lama. Migration `ADD COLUMN IF NOT EXISTS` untuk safe-apply ke DB existing. Field `amount` legacy diterima sebagai alias `total_amount`.
  - **Tabel `nomination_epb_line_items` terpisah:** Line items bisa multi-row per EPB (STS Fee + 7+ kemungkinan Biaya Jasa Tambahan). Normalize ke tabel anak agar query/index efisien dan FK cascade rapi. `volume`/`rate` disimpan sebagai string display (bukan numeric) karena STS yang format — LPS tidak menghitung ulang.
  - **PPn rate disimpan, bukan hardcode 11%:** Field `vat_rate` (default 0.11) di `nomination_epb` agar UI bisa render label "PPn (11%)" dinamis. Mencegah hardcoded yang akan rusak saat regulasi PPn berubah. STS tetap yang otoritatif menentukan rate.
  - **`No Rek` + `Kode Bayar` font-mono tanpa copy button awal:** Pattern font-mono cukup memudahkan customer membaca & menyeleksi. Tombol copy-to-clipboard ditahan dulu (bisa ditambah belakangan kalau ada feedback) untuk menjaga UI tidak terlalu sibuk. Konsisten dengan pattern Section Card existing.
  - **Header EPB di M9b sederhanakan:** Sebelumnya header detail menampilkan vessel name ("MV Nusantara Star") yang sudah ada di vessel_ops grid. Dihapus dari header untuk mengurangi duplikasi. Kini header = EPB number + nomination ref + status badge + Download button.
  - **4-file rule intact:** Tidak ada penambahan file di `module/`. Semua artefak baru ditempatkan di lokasi yang tepat (design system, per-modul UI doc, handoff).
- **Results / Next Steps:**
  - BRD v3.4 + design system v1.3 + handoff v2.1 (M9b) / v2.2 (M9) sinkron full pipeline.
  - Replit Agent dapat langsung implement dengan urutan: (1) M9 migrasi schema (ALTER + tabel baru) → (2) M9 webhook handler extended → (3) M9b endpoint detail JOIN + Download proxy → (4) Frontend `formatCurrency` helper → (5) Invoice Detail Card komponen shared → (6) M9b EPBDetailPage rewrite → (7) M9 NominationStatusPage EPB section.
  - **Dependency eksternal:** STS Platform perlu kirim payload extended (vessel_ops, line_items, subtotal, vat_*, bank_info, epb_pdf_url) di webhook APPROVED. Koordinasi dengan tim STS dibutuhkan sebelum deploy E2E. LPS bisa deploy duluan dengan UI fallback minimal aktif.
  - File yang **belum** diupdate karena tidak terdampak: M1–M8, M9c, M10–M12.

## 2026-05-14T00:00:00+07:00
- **Task Performed:** Split modul EPB & Invoice menjadi dua modul terpisah (M9b EPB + M9c Invoice baru). Tambah partial payment EPB (min 1 USD ekuivalen IDR), sync status labels ke production (Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas), tambah kategori filter "Perlu Tindakan" + Overdue indicator. BRD naik ke v3.3, design system naik ke v1.2. Implementasi end-to-end: BRD → BRD per modul → module/ → implementation/design/ → replit-handoff. Dipicu oleh screenshot flow production yang menunjukkan EPB dan Invoice sebagai dua flow terpisah dengan decision logic shortfall+additional service.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — bump ke v3.3. Tambah change log entries #21–#26. Revisi section 2.1 (9b) ke EPB-only scope dengan partial payment logic. **Tambah section 2.1 (9c)** baru: Invoice (Customer Portal) dengan source EPB_SHORTFALL / ADDITIONAL_SERVICE. Update tabel section 3.1 (tambah M9c). Update tabel role & access section 3.3. Revisi FR-EI-01..09 ke label baru + tambah FR-EI-10 (partial payment). **Tambah section 3.4.9c** baru: FR-IN-01..08. Update Appendix A.2 dengan dua sub-section (EPB / Invoice). Update tabel section 1.3 pisah baris EPB & Invoice Customer Portal jadi dua.
  - `document/brd/m9b-epb-invoice.md` — rewrite penuh. Judul jadi "EPB (Customer Portal)" (scope EPB only). Tambah partial payment business rules, webhook EPB_SHORTFALL_DETECTED, status labels v3.3, FR-EI-10. Catatan v3.3 di top.
  - `document/brd/m9c-invoice.md` — **file baru**. BRD per modul untuk M9c dengan source rules, 4 status, 4 webhook events, FR-IN-01..08.
  - `document/brd/README.md` — index diupdate: M9b rename label jadi "EPB", tambah baris M9c.
  - `CLAUDE.md` — tabel BRD per modul diupdate (M9b rename + M9c baru).
  - `implementation/design/lps-design-system.md` — bump ke v1.2. Status palette tambah: "Action Required (Perlu Tindakan)" rose + "Overdue indicator" rose deeper. Status mapping LPS direvisi: UNPAID→"Belum Dibayar", Menunggu Verifikasi Pembayaran→"Menunggu Verifikasi", Pembayaran Dikonfirmasi→"Lunas". Tambah meta-kategori "Perlu Tindakan" (UNPAID+PAYMENT_REJECT+Overdue). Tambah 3 komponen baru di §3.1: **KPI Filter Pill** (3 default kategori billing), **Overdue Date Display** (deteksi due_date < now), **Inline Info Banner** (4 varian per status untuk list items). Update inline example voyage card pakai label "Lunas". Changelog v1.2.
  - `module/epb-invoice/README.md` — rewrite. Judul jadi "M9b — EPB". Scope EPB-only, partial payment, link ke M9c, dual menu UI explanation.
  - `module/epb-invoice/requirements.md` — rewrite. FR-EI-01..09 dengan label v3.3 + FR-EI-10 baru (partial payment + shortfall trigger). Tambah NFR "Currency Conversion".
  - `module/epb-invoice/specifications.md` — rewrite. Lifecycle diagram baru dengan paid_amount + shortfall path. Section "Partial Payment Logic" baru. DB schema tambah `total_amount`, `paid_amount`, `currency`, `due_date`, `shortfall_pushed`. Webhook handler updated (3 events: REJECT/PAID/SHORTFALL_DETECTED) dengan idempotency. Upload endpoint validasi paid_amount min IDR. Summary endpoint baru. Routes pindah ke `/customer/billing/epb`.
  - `module/epb-invoice/user-stories.md` — rewrite. US-EI-01..06 dengan label v3.3, US-EI-03 includes nominal input flow + helper text. **US-EI-07 baru:** Partial Payment Triggers Invoice (cross-link ke M9c).
  - `module/invoice/README.md` — **file baru**. M9c overview, dependencies, downstream (cross-link ke M9b/M10), key decisions termasuk "Invoice tidak block nominasi".
  - `module/invoice/requirements.md` — **file baru**. FR-IN-01..08 + NFR Idempotency.
  - `module/invoice/specifications.md` — **file baru**. Lifecycle + DB schema `invoices` + `invoice_payment_proofs` dengan source constraint (CHECK), 4 STS webhook events, internal service `CreateFromShortfall/CreateFromAdditionalService`, upload endpoint tanpa nominal field, service key mapping ke label ID (7 service keys).
  - `module/invoice/user-stories.md` — **file baru**. US-IN-01..07 termasuk source-specific detail rendering + automatic creation dari webhook.
  - `implementation/design/m9b-epb-invoice-ui.md` — rewrite penuh. Top tabs EPB|Invoice pattern. KPI Filter Pills + Tabs filter Semua/Perlu Tindakan/Menunggu Verifikasi/Lunas. Table dengan Overdue Date Display. Bayar form dengan nominal input + validation (`>= minIDR` and `<= total_amount`). Cross-link section ke Invoice saat Lunas dengan shortfall. Routes `/customer/billing/epb`.
  - `implementation/design/m9c-invoice-ui.md` — **file baru**. List page dengan Source filter Select + Source badge column (Shortfall EPB / Additional Service). Detail page dengan Asal Tagihan section source-specific (link ke parent EPB OR list service labels). Bayar form **TANPA nominal input** (full payment only). Service key mapping ke label ID.
  - `implementation/design/m9-nomination-status-payment-ui.md` — header v3.3 sync note. Status banner labels diupdate. Route ke M9b berubah dari `/customer/epb-invoice/:id` → `/customer/billing/epb/:id`. Inline reference UNPAID → "Belum Dibayar".
  - `implementation/design/m10-customer-dashboard-ui.md` — header v3.3 sync note. Routes terkait section diupdate: tambah `/customer/billing`, `/customer/billing/epb`, `/customer/billing/invoice`. Filter nominasi aktif hapus referensi EPB_CONFIRMATION_SUBMITTED yang sudah dihapus.
  - `implementation/replit-handoff/m9b-epb-invoice.md` — bump ke v2.0. Version history. Full rewrite: scope EPB only, partial payment endpoint dengan paid_amount validation, currency conversion logic (`USD_IDR_RATE` + fallback `MIN_PAYMENT_IDR_FALLBACK`), webhook `EPB_SHORTFALL_DETECTED` handler memforward call ke M9c `invoiceService.CreateFromShortfall(...)`. `EPB_PAID` handler update `nominations.status = PAYMENT_CONFIRMED` (parsial OK) + conditional shortfall handling. Migration tambah kolom (IF NOT EXISTS safe). Step 7 `BillingLayout` shared component dengan top tabs EPB|Invoice. Step 8-9 Frontend dengan KPI Pills + status labels v3.3.
  - `implementation/replit-handoff/m9c-invoice.md` — **file baru**. Full handoff dengan migration `invoices`+`invoice_payment_proofs` (source constraint check). InvoiceService `CreateFromShortfall` & `CreateFromAdditionalService`. STS webhook handler 4 events dengan idempotency. Upload endpoint tanpa `paid_amount` field. Detail page Asal Tagihan section conditional render. SERVICE_LABELS constant.
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — bump ke v2.1. Note v3.3 changes di header. Route `/customer/epb-invoice` diganti `/customer/billing/epb` (replace_all). Status mapping line direvisi ke label v3.3. M9 sekarang juga set `total_amount`, `currency`, `due_date` saat insert epb_payments row (kolom baru di M9b v2.0 migration).
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md` — statusLabels map: "Menunggu Verifikasi Pembayaran"→"Menunggu Verifikasi", "Pembayaran Dikonfirmasi"→"Lunas". Status badge color line label diupdate. Sidebar nav "EPB & Invoice" link berubah ke `/customer/billing` (v3.3 note).
- **Logic / Decisions Made:**
  - **Module split rationale:** EPB dan Invoice secara business punya properti berbeda (EPB partial OK + bisa generate Invoice; Invoice full payment + tidak block nominasi). Memisah backend & module structure mencegah kebingungan scope. UI tetap satu menu untuk konsistensi UX customer.
  - **Folder `module/epb-invoice/` tidak di-rename** ke `module/epb/` untuk menjaga git history & semua cross-reference existing (CLAUDE.md, BRD per modul filename, replit-handoff filename). Konten dan judul markdown diadjust ke "EPB only", tapi filename tetap.
  - **Partial payment min 1 USD ekuivalen IDR:** Threshold dihitung dari `USD_IDR_RATE` system config (refresh harian dari STS) dengan fallback `MIN_PAYMENT_IDR_FALLBACK` saat kurs stale/unavailable. Validasi di client (UX) DAN server (security). `paid_amount` snapshot per attempt di `epb_payment_proofs` (audit trail).
  - **Shortfall trigger arsitektur:** STS push `EPB_SHORTFALL_DETECTED` ke M9b. M9b handler memforward ke M9c `invoiceService.CreateFromShortfall(...)` internal call. STS bisa kirim event terpisah atau gabung dengan `EPB_PAID` payload. Idempotency via `shortfall_pushed` flag di `epb_payments` + `sts_idempotency_key` unique di `invoices`.
  - **Invoice tidak block nominasi:** Confirmed business decision — `nominations.status = PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (parsial OK). Invoice = settlement separate yang bisa berjalan paralel/setelah voyage. Voyage tidak menunggu Invoice. Tercermin di handler `INVOICE_PAID` yang TIDAK menyentuh `nominations.status`.
  - **Status labels production sync:** "UNPAID"→"Belum Dibayar", "Pembayaran Dikonfirmasi"→"Lunas", "Menunggu Verifikasi Pembayaran"→"Menunggu Verifikasi". Token DB tetap pakai SCREAMING_SNAKE_CASE untuk konsistensi backend; mapping ke label UI di frontend constant. Tambah meta-kategori "Perlu Tindakan" sebagai filter (bukan status DB) untuk UX better grouping (UNPAID + PAYMENT_REJECT + Overdue).
  - **Card-list vs table:** User explicitly pilih pertahankan table di list page (override pattern production yang pakai card-list) — dokumentasi rasional ada di m9b UI doc. Hanya 4 KPI pill yang adopsi pattern dari production untuk header summary.
  - **Routes pakai `/customer/billing/...`:** Single base URL untuk EPB+Invoice sesuai keputusan user. Landing `/customer/billing` redirect ke `/customer/billing/epb` (default tab). Sidebar tetap satu item "EPB & Invoice" pointing ke `/customer/billing`. Top tabs EPB|Invoice di setiap halaman billing dengan URL change saat switch tab (preserve bookmark/back).
  - **Invoice supports re-upload:** Confirmed — `PAYMENT_REJECT` state di Invoice loop seperti EPB. Tidak ada 3-status-only varian.
  - **Overdue indicator:** Computed field `is_overdue` di API response (server-side: `due_date < now() AND status != PAID`). UI tampilkan pattern Overdue Date Display (rose-700 + AlertTriangle). Tabs filter "Perlu Tindakan" include overdue dalam logic.
  - **4-file rule intact:** `module/epb-invoice/` tetap 4 file (README/requirements/specifications/user-stories). `module/invoice/` baru juga 4 file. Tidak ada extra file di module folders. Verified via `ls`.
- **Results / Next Steps:**
  - BRD v3.3 sinkron dengan production flow + screenshot terbaru. Pipeline implementasi end-to-end siap dieksekusi ke Replit Agent: M9b v2.0 handoff (revisi + migrasi schema) → M9c handoff (baru).
  - Urutan implementasi yang direkomendasikan: M9b migrasi schema (alter table tambah kolom) → M9b backend (partial payment + webhook shortfall) → **M9c migration + backend + service** → M9b frontend (BillingLayout shared + EPB pages) → M9c frontend (Invoice pages). M9c bergantung pada M9b InvoiceService callable, jadi M9c skeleton harus exist sebelum M9b deploy.
  - File yang **belum** diupdate karena tidak terdampak: M7, M8, M11, M12 dokumen.
  - Cross-link UI (M9b detail → M9c detail saat shortfall) sudah didokumentasikan di design + handoff. Saat Replit deploy, perlu test E2E flow: bayar EPB parsial → terima webhook PAID + SHORTFALL → check Invoice muncul di tab Invoice → bayar Invoice → check Lunas.

## 2026-05-13T23:00:00+07:00
- **Task Performed:** Update design system ke v1.1 dan M7 UI doc untuk pattern login customer yang baru (split-screen + image showcase carousel). Dipicu oleh screenshot login baru dari user dengan layout split-screen (form kiri + image kapal kanan + carousel dots + caption "Keselamatan & Keandalan").
- **Files Modified:**
  - `implementation/design/lps-design-system.md` — bump ke v1.1. Tambah: (1) section 2.10 "Brand mark" dengan dua varian (wordmark inline + stacked) + canonical naming "LPS System"; (2) Input with leading icon pattern di §3.1 (Mail icon kiri untuk email, Lock + eye toggle untuk password, background subtle `bg-slate-50/60`); (3) Divider with text pattern; (4) Auth Split-Screen layout di §3.4 (50/50 split, brand mark top-left, form centered vertikal max-w-md, image showcase kanan, breakpoint lg); (5) Image Showcase Carousel pattern dengan caption overlay bottom-left + dots indicator bottom-right; (6) 3 default slides Surface A (Keselamatan & Keandalan, Integrasi STS Real-time, Pantauan 24/7) verbatim; (7) Auth Centered layout untuk admin login (minimal, no showcase). Changelog v1.1 ditambah.
  - `implementation/design/m7-customer-authentication-ui.md` — header bump v1.1. Section 3.1 Customer Login rewrite total: ganti centered card → Auth Split-Screen. Heading diubah dari "Portal Pelanggan LPS" → "Selamat Datang" (sesuai screenshot). Field pattern diganti ke Input with leading icon. Tambah divider "atau" + link "Daftar di sini" bold. Footer copy verbatim dari screenshot. Section 3.2 Customer Register: tambah catatan layout (split-screen sama dengan login, form panel scrollable, max-w-xl, image showcase tetap fixed). Heading register: "Daftar Akun Pelanggan". Section 4.1 Admin Login: pakai Auth Centered layout dengan brand mark stacked, English copy, no showcase. Section 5 Component Usage Summary diupdate dengan 4 komponen baru.
  - `implementation/replit-handoff/m7-customer-authentication.md` — section "Prerequisites — Design Reference (WAJIB)" diupdate. Tambah tabel layout pattern per halaman auth (split-screen vs centered). Tambah "Required komponen baru (v1.1)" yang menyebut Input with leading icon, Image Showcase Carousel, Brand mark wordmark inline, Divider with text. Heading + footer copy verbatim per halaman.
- **Logic / Decisions Made:**
  - **Customer login pakai Auth Split-Screen** sesuai screenshot, customer register juga (konsisten visual). Admin login tetap Auth Centered karena halaman fungsional internal — marketing showcase tidak relevan.
  - **Carousel 3 default slides** ditambahkan ke design system sebagai tabel referensi (bukan hanya placeholder). Konten: Keselamatan & Keandalan (verbatim dari screenshot), Integrasi STS Real-time, Pantauan 24/7. Disimpan sebagai config di `src/config/authSlides.ts` agar mudah di-update tanpa rebuild design.
  - **Input with leading icon ditambahkan sebagai pattern terpisah** di §3.1 — beda dari Input base (background tinted `bg-slate-50/60`, padding lebih besar `py-3`, icon absolute kiri). Ini pattern khusus auth, bukan untuk form dalam app yang pakai Input base.
  - **Brand mark wordmark inline** (anchor + "LPS System") ditegaskan sebagai canonical untuk halaman auth. Brand mark stacked (Ship + brand 2-line) untuk sidebar. Sebelumnya design system belum dokumentasikan dua varian eksplisit.
  - **Canonical naming "LPS System"** untuk display — bukan "LPS Platform" (yang hanya untuk dokumentasi internal/BRD). Konsisten dengan screenshot.
  - **Form register tetap pakai split-screen** meski lebih panjang dari login: form panel scrollable, image showcase kanan fixed. Form max-w-md → max-w-xl untuk register (field tidak terlalu sempit).
  - **Modul lain tidak perlu update** karena perubahan ini terbatas di pattern auth (M7 only). Modul M8/M9/M9b/M10/M12 tidak punya halaman auth.
- **Results / Next Steps:**
  - Design system kini punya pattern auth eksplisit (split-screen + carousel + centered) yang dipisahkan dari layout app utama.
  - M7 implementation handoff sekarang lebih akurat — Replit Agent bisa langsung implement login dengan pattern yang match production.
  - Jika ada modul auth baru di masa depan (mis. password reset, 2FA setup), pattern yang sama bisa direuse dari design system §3.4.

## 2026-05-13T22:00:00+07:00
- **Task Performed:** Integrasi skill ui-ux-pro-max ke seluruh pipeline dokumentasi LPS. Bikin design system terpusat (Professional Maritime style) + per-modul UI design doc untuk 6 modul, tambah policy di CLAUDE.md & README.md, dan update 6 replit-handoff dengan section "Prerequisites — Design Reference". Style + komponen disesuaikan dengan 7 screenshot production yang diberikan user (Customer Portal Surface A + LPS System Bunati Port Surface B).
- **Files Modified:**
  - `implementation/design/lps-design-system.md` — **file baru**: master design system 7 section (Foundation, dua Surface preset, Component Library lengkap dengan Tailwind class snippets, Responsive strategy, DoD, Cross-refs, Changelog). Status palette mapping verbatim ke status business LPS (Menunggu Review, Disetujui, Perlu Revisi, Menunggu Verifikasi Pembayaran, Pembayaran Dikonfirmasi, UNPAID, Draft, WARNING).
  - `implementation/design/m7-customer-authentication-ui.md` — **file baru**: dual surface (A customer register/login + B admin Customer Management/detail/Add).
  - `implementation/design/m8-nomination-submission-ui.md` — **file baru**: form 5 section (Kapal, Voyage, Cargo, Additional Service 7 checkbox, Dokumen Pendukung dual-tab).
  - `implementation/design/m9-nomination-status-payment-ui.md` — **file baru**: detail page 2-column, status banner 6 varian (Draft/Submitted/Approved/Need Revision/Waiting Verification/Confirmed), timeline custom, EPB Detail section.
  - `implementation/design/m9b-epb-invoice-ui.md` — **file baru**: list page 5 tabs filter, detail 2-column, status banner 4 payment status, bank account card dengan copy clipboard, payment history nested.
  - `implementation/design/m10-customer-dashboard-ui.md` — **file baru**: 5 halaman (Dashboard, Nominasi list, Voyage tracking dengan Leaflet, Document Master append-only, Cuaca & Alert) — layout langsung disalin dari production screenshot.
  - `implementation/design/m12-system-configuration-ui.md` — **file baru**: Surface B operator, 9 halaman settings (Overview, Users, Roles matrix, System Config tabs, API Integrations, Equipment, Audit Log immutable, STS Sync dengan Recharts).
  - `CLAUDE.md` — tambah section "UI/UX Standard — WAJIB untuk semua kerja UI" (7 aturan: baca design system + invoke ui-ux-pro-max + dua surface mapping + status badge variant + tech stack + per-modul design doc + saat ada pattern baru + replit handoff Prerequisites). Tambah Placement Rules row.
  - `README.md` — tambah section design system di `/implementation/design/` description + aturan ringkas UI/UX.
  - `module/customer-authentication/README.md` — append section "UI/UX Design" dengan link ke design system + m7-ui doc + surface info.
  - `module/nomination-submission/README.md` — append section UI/UX Design (Surface A).
  - `module/nomination-status-payment/README.md` — append section UI/UX Design (Surface A).
  - `module/epb-invoice/README.md` — append section UI/UX Design (Surface A).
  - `module/customer-dashboard-monitoring/README.md` — append section UI/UX Design (Surface A).
  - `module/system-configuration/README.md` — append section UI/UX Design (Surface B).
  - `implementation/replit-handoff/m7-customer-authentication.md` — tambah section "Prerequisites — Design Reference (WAJIB)" sebelum Tech Stack, instruksi baca 2 file (design system + m7-ui), surface mapping dual, UI rules ringkas (shadcn, status badge variant, color tokens, card pattern, font, icon).
  - `implementation/replit-handoff/m8-nomination-submission.md` — tambah section Prerequisites Design Reference Surface A.
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — tambah section Prerequisites Design Reference Surface A dengan emphasis status banner & timeline.
  - `implementation/replit-handoff/m9b-epb-invoice.md` — tambah section Prerequisites Design Reference Surface A dengan emphasis 4 payment status & bank account card.
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md` — tambah section Prerequisites Design Reference Surface A dengan emphasis Leaflet map + Recharts + Document Master append-only.
  - `implementation/replit-handoff/m12-system-configuration.md` — tambah section Prerequisites Design Reference Surface B dengan emphasis sidebar nested, top bar, permission-based rendering, immutable audit log.
- **Logic / Decisions Made:**
  - **Dua surface system**, bukan satu — observasi dari 7 screenshot: Customer Portal (Surface A, ID, sidebar simpel, card-heavy, no top bar) berbeda jelas dengan LPS System Bunati Port (Surface B, EN, sidebar nested collapsible, top bar persistent dengan search/datetime/notifications/profile, data-dense KPI+map+chart). Design system mendefinisikan keduanya sebagai preset turunan dari Foundation yang sama.
  - **Modul M7 unik karena dual-surface**: customer side (register/login) pakai Surface A, admin side (Customer Management) pakai Surface B. m7-customer-authentication-ui.md dibagi 2 section eksplisit.
  - **Style line:** Professional Maritime — minimalism + flat dengan jangkar navy `#0B2545`/`#0F2A4D` (sidebar, primary), canvas `bg-slate-50`, card `bg-white rounded-2xl shadow-sm`. TIDAK pakai glassmorphism, gradient besar, atau heavy shadow.
  - **Status badge wajib soft pill pattern** (50/200/700 dengan border halus) — bukan solid color. Mapping label business ke variant didefinisikan eksplisit di design system §2.1 sehingga konsisten lintas modul.
  - **Surface B chart:** Recharts dengan threshold coloring (green safe / yellow warning / red danger) sesuai screenshot weather forecast. Map: Leaflet + OpenStreetMap dengan custom marker (Lucide Navigation rotated untuk vessel, Anchor untuk anchor point).
  - **Policy enforcement triple-layer** (sesuai pilihan user): CLAUDE.md (project-level rule wajib) + README.md (navigasi + ringkas) + module/<name>/README.md (per-modul pointer). Mencegah agent skip baca design system di pekerjaan UI ke depan.
  - **Replit handoff Prerequisites:** Section disisipkan SEBELUM Tech Stack agar urutan baca natural — design reference dulu, baru tech stack, baru step-by-step. Link relatif ke design doc (`../design/...`).
  - **4-file rule module/ intact:** Semua perubahan di module/<name>/ hanya append section ke file README.md yang sudah ada — tidak menambah file baru. Sudah diverify dengan `ls`.
  - **Tidak meng-update file existing `2026-04-25-module7-decomposition-design.md`** karena itu design decomposition (bukan UI design) dengan scope berbeda.
  - **Modul yang dikerjakan: hanya yang punya module/ folder** sesuai pilihan user — M7, M8, M9, M9b, M10, M12. M11 (Monitoring Dashboard internal) belum punya module/ folder jadi di-skip; jika dikerjakan, surface-nya akan Surface B juga.
- **Results / Next Steps:**
  - LPS Platform sekarang punya design system terpusat yang siap pakai. Total 7 file design baru + 9 file existing diupdate = 16 file modified.
  - Setiap AI agent / Replit Agent yang melanjutkan kerja UI akan menemukan design reference yang clear di 3 titik: project CLAUDE.md, module README, dan replit handoff Prerequisites.
  - Tidak ada konflik dengan pipeline existing — semua modul yang sudah punya replit-handoff tetap valid; section Prerequisites baru bersifat additive.
  - **Saat M11 atau modul UI baru ditambahkan ke pipeline:** bikin `implementation/design/m<N>-<name>-ui.md` baru mengikuti struktur sama (Ringkasan, Page Inventory, halaman per route, Component Usage, Edge Cases, Cross-refs), update Changelog di lps-design-system.md jika ada pattern baru.

## 2026-05-11T02:00:00+07:00
- **Task Performed:** Jalankan implementation pipeline M12 — System Configuration. Buat 4 file module/ dan 1 file replit-handoff dari nol, menggunakan BRD M12 sebagai scope aktif dan old-data (Master Data lama) sebagai referensi implementasi yang sudah berjalan.
- **Files Modified:**
  - `module/system-configuration/README.md` — **file baru**: overview, in/out scope, dependency matrix, key decisions (data master di STS Platform, local cache, warisan dari old-data)
  - `module/system-configuration/requirements.md` — **file baru**: FR-SC-01 s/d FR-SC-06, tabel cakupan per sub-modul, daftar 9 system roles, cross-cutting NFR
  - `module/system-configuration/specifications.md` — **file baru**: 6 section teknis — (1) User & Role Management: business rules, DB schema (roles, admin_users, role_permissions), API endpoints; (2) System Configuration: business rules, DB schema (system_configs, api_configs, alert_configs), API endpoints; (3) Equipment Management: business rules, DB schema (equipment, equipment_status_history), API endpoints; (4) Audit Log: business rules, DB schema (audit_logs + indexes), API endpoint; (5) STS Data Sync: business rules, DB schema (sts_vessels_cache, sts_sync_log), API endpoints; (6) Frontend routes
  - `module/system-configuration/user-stories.md` — **file baru**: US-SC-01 s/d US-SC-05 (User & Role, Permission Matrix, System & API Config, Equipment, Audit Log & STS Sync) dengan acceptance criteria testable
  - `implementation/replit-handoff/m12-system-configuration.md` — **file baru**: 14 step eksekusi — Step 1 migration (IF NOT EXISTS, catatan cek tabel yang sudah ada), Step 2 seed data (roles, system_configs, api_configs, alert_configs), Step 3 Go models, Step 4 audit log helper, Step 5 encryption helper (AES-256-GCM), Step 6 handlers (8 grup endpoint), Step 7 settings layout sidebar, Step 8–14 frontend pages (User, Role/Permission, SystemConfig, ApiIntegrations, Equipment, AuditLog, StsSync) + acceptance checklist 20 item
- **Logic / Decisions Made:**
  - **Referensi old-data digunakan sebagai baseline:** System roles (9 role), business rules user (BR-SC-01 s/d 06), equipment types dan status, API integration names — semua diwarisi dari implementasi admin yang sudah berjalan. Pipeline baru tidak mengubah, hanya melengkapi dan memformalisasi.
  - **Scope M12 sudah dikurangi dari old-data:** Vessel Master, Stakeholder Master, Zone, Anchor Point, Rate Card semua OUT of scope M12 — sudah di STS Platform. M12 hanya: user/role, system config, equipment, audit log, STS sync.
  - **Migration pakai IF NOT EXISTS:** Handoff menegaskan agar Replit Agent cek tabel yang sudah ada sebelum menjalankan migration, menghindari conflict dengan tabel admin yang mungkin sudah ada.
  - **API key masking:** Tidak pernah return plaintext credential dari API; `MaskValue()` helper dibuat eksplisit; field kosong saat update = tidak overwrite.
  - **Audit log immutable:** Tidak ada endpoint POST/PUT/DELETE untuk audit_logs selain write dari handler internal.
- **Results / Next Steps:**
  - Pipeline M12 lengkap: module/ (4 file) + replit-handoff (1 file).
  - Siap dihandoff ke Replit Agent. Replit Agent harus baca Prerequisites dan catatan "Penting: Cek tabel yang sudah ada" di Step 1 sebelum menjalankan migration.

## 2026-05-11T01:00:00+07:00
- **Task Performed:** Sinkronisasi fitur Additional Service M8 ke seluruh implementation pipeline: module/ (3 file) + replit-handoff.
- **Files Modified:**
  - `module/nomination-submission/requirements.md` — sisipkan FR-NS-03 (Additional Service); renumber FR-NS-03→04, 04→05; update FR-NS-05→06 dengan keterangan payload STS.
  - `module/nomination-submission/specifications.md` — tambah Section 2 "Additional Service" (tabel 7 service keys, behavior, DB); renumber section 2–7 menjadi 3–8; tambah tabel `nomination_additional_services` di DB schema; tambah field `additional_services` di STS payload.
  - `module/nomination-submission/user-stories.md` — update AC US-NS-01 (tambah checkbox AC); sisipkan US-NS-02 baru "Select Additional Service" dengan 5 AC; renumber US-NS-02→US-NS-03, US-NS-03→US-NS-04.
  - `implementation/replit-handoff/m8-nomination-submission.md` — (1) Migration: tambah tabel `nomination_additional_services`; (2) Model: tambah `NominationAdditionalService` struct + relasi di `Nomination`; (3) Repository: tambah method `SetAdditionalServices`; (4) Handler: tambah field `additional_services` di request body, tambah validasi service key, panggil `SetAdditionalServices` di draft dan submit flow; (5) STS payload: tambah field `additional_services`; (6) Frontend: tambah section checkbox 7 pilihan di NominationFormPage; (7) Acceptance checklist: tambah 5 item terkait Additional Service.
- **Logic / Decisions Made:**
  - Additional Service disimpan di tabel terpisah `nomination_additional_services` (many-to-many dengan UNIQUE constraint), bukan sebagai array column di tabel `nominations` — lebih query-friendly dan tidak bergantung pada tipe array PostgreSQL.
  - `SetAdditionalServices` menggunakan delete-then-insert dalam satu transaksi untuk mendukung edit Draft yang mengubah pilihan.
  - Validasi service key dilakukan di backend pada saat submit untuk mencegah nilai invalid masuk ke DB atau payload STS.
  - Payload STS selalu menyertakan `additional_services` (bisa `[]`) agar STS Platform tidak perlu handle field absent.
- **Results / Next Steps:**
  - Semua 4 layer pipeline M8 (BRD → module → replit-handoff) kini sinkron untuk fitur Additional Service.
  - Siap dihandoff ke Replit Agent.

## 2026-05-11T00:00:00+07:00
- **Task Performed:** Tambah fitur Additional Service (opsional, multi-select) ke M8 — Nomination Request Submission. Sinkronisasi ke BRD utama dan file BRD per modul.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — (1) Section 2.1 M8 Cakupan: tambah bullet Additional Service dengan 7 pilihan; (2) Section 3.4.8: tambah FR-NS-03 baru (Additional Service), renumber FR-NS-03→FR-NS-04, FR-NS-04→FR-NS-05, update FR-NS-05→FR-NS-06 (tambah keterangan "termasuk daftar Additional Service yang dipilih").
  - `document/brd/m8-nomination-submission.md` — Cakupan dan tabel FR diupdate verbatim sesuai BRD utama.
- **Logic / Decisions Made:**
  - Additional Service bersifat opsional: customer boleh tidak memilih sama sekali, boleh memilih lebih dari satu.
  - Daftar 7 pilihan: Tank Cleaning, Pengisian Bahan Bakar atau Air Bersih (Bunkering & Fresh Water Supplying), Short Stay Temporary, Supply Logistic, Lay Up, Ship Chandler, Kapal Emergency.
  - FR-NS-03 baru disisipkan di antara FR-NS-02 dan FR-NS-03 lama (Draft) sehingga alur logis form → additional service → draft → submit → generate → kirim ke STS.
  - FR-NS-06 (sebelumnya FR-NS-05) diupdate untuk menegaskan bahwa payload ke STS Platform menyertakan Additional Service yang dipilih.
- **Results / Next Steps:**
  - BRD utama dan `document/brd/m8-nomination-submission.md` kini sinkron.
  - Layer `module/nomination-submission/` dan `implementation/replit-handoff/m8-nomination-submission.md` belum diupdate — perlu disinkronkan jika pekerjaan dilanjutkan ke tahap implementasi.

## 2026-05-09T15:00:00+07:00
- **Task Performed:** Review menyeluruh semua 13 file BRD per modul di `document/brd/` dan perbaiki 5 ketidakkonsistenan yang ditemukan dibanding BRD utama.
- **Files Modified:**
  - `document/brd/m9-nomination-status.md` — **rewrite Cakupan**: sebelumnya hanya ada sub-header kosong tanpa isi; sekarang diisi verbatim dari BRD utama Section 2.1 M9 (Status Tracking + EPB Detail + Batasan Modul + blockquote). Role & Access juga dilengkapi dengan semua role (sebelumnya hanya 3 role).
  - `document/brd/m7-customer-authentication.md` — (1) Cakupan: baris terakhir dipotong, tambah "(NPWP, NIP, Company Profile)" sesuai BRD utama; (2) FR-CA-01: kalimat dipotong, tambah "dan tidak ditampilkan sebagai field input di form registrasi" sesuai BRD utama.
  - `document/brd/m8-nomination-submission.md` — Cakupan: hapus baris "Pilihan upload dokumen..." (tidak ada di BRD utama Section 2.1 M8; itu konten dari FR-CD-07 M10). Hapus section "Out of Scope" (tidak ada di BRD utama untuk M8).
  - `document/brd/m11-monitoring-dashboard.md` — (1) Definisi: dipotong menjadi satu kalimat ringkas; sekarang verbatim dari BRD utama; (2) Cakupan baris terakhir: "Customer mendapat view terbatas" → "Pemilik Kapal mendapat view terbatas sesuai hak akses masing-masing" sesuai BRD utama; (3) Hapus section "Out of Scope" dan ganti dengan blockquote Catatan verbatim dari BRD utama.
  - `document/brd/m12-system-configuration.md` — (1) Definisi: dipotong 1 kalimat; tambah "Modul ini merupakan sisa dari Master Data & Configuration sebelumnya, setelah data master entity (vessel, stakeholder, rate card) dipindahkan ke STS Platform"; (2) STS Platform API: tambah **bold** sesuai BRD utama; (3) Catatan: format diubah ke blockquote + tambah kalimat terakhir tentang sync registrasi customer ke STS Platform.
- **Logic / Decisions Made:**
  - Prinsip: konten di `document/brd/` harus **verbatim** dari BRD utama — tidak boleh diparafrase, dipotong, atau ditambah konten yang tidak ada di BRD utama.
  - M8 "Out of Scope" dan M11 "Out of Scope" adalah konten tambahan yang tidak ada di BRD utama — dihapus.
  - M8 "Pilihan upload dokumen" adalah konten yang terambil dari FR-CD-07 M10 secara tidak tepat — dihapus.
- **Results / Next Steps:**
  - Semua 13 file BRD per modul kini verbatim dan konsisten dengan BRD utama.
  - Tidak ada perbedaan substansial antara `document/brd/` dan `document/BRD-LPS-V3-Analysis.md`.

## 2026-05-09T14:00:00+07:00
- **Task Performed:** Membuat folder `document/brd/` berisi 13 file BRD per modul (M1–M12 + index README). Mengupdate `README.md` dan `CLAUDE.md` dengan aturan Dual-Update dan daftar file per modul.
- **Files Modified:**
  - `document/brd/README.md` — **file baru**: index seluruh file per modul, aturan sync, struktur setiap file
  - `document/brd/m1-vessel-monitoring.md` — **file baru**: BRD M1
  - `document/brd/m2-vessel-communication.md` — **file baru**: BRD M2
  - `document/brd/m3-government-integration.md` — **file baru**: BRD M3
  - `document/brd/m4-incident-emergency.md` — **file baru**: BRD M4
  - `document/brd/m5-weather-monitoring.md` — **file baru**: BRD M5
  - `document/brd/m6-radar-navigation.md` — **file baru**: BRD M6
  - `document/brd/m7-customer-authentication.md` — **file baru**: BRD M7 (lengkap dengan link replit-handoff)
  - `document/brd/m8-nomination-submission.md` — **file baru**: BRD M8
  - `document/brd/m9-nomination-status.md` — **file baru**: BRD M9 (termasuk tabel STS webhook)
  - `document/brd/m9b-epb-invoice.md` — **file baru**: BRD M9b (termasuk flow diagram 4 status)
  - `document/brd/m10-customer-dashboard.md` — **file baru**: BRD M10
  - `document/brd/m11-monitoring-dashboard.md` — **file baru**: BRD M11
  - `document/brd/m12-system-configuration.md` — **file baru**: BRD M12
  - `README.md` — tambah section `/document/brd/` dengan Dual-Update Invariant rule
  - `CLAUDE.md` — update directory architecture, tambah section "BRD Per Modul — Dual-Update Rule", update Foundational Documents
- **Logic / Decisions Made:**
  - **Sumber kebenaran tetap satu:** `BRD-LPS-V3-Analysis.md` adalah primary source of truth. File di `document/brd/` adalah *derived views* — bukan duplikasi independen.
  - **Dual-Update Invariant:** setiap kali BRD utama diupdate, file `brd/<modul>.md` yang terkait harus diupdate dalam sesi yang sama. Aturan ini didokumentasikan di CLAUDE.md, README.md, dan `document/brd/README.md` agar AI agent berikutnya memahaminya.
  - **Konten setiap file:** Definisi, Tujuan, Cakupan, FR table, Role & Access, cross-reference ke BRD utama + module/ + replit-handoff (jika ada). Tidak ada konten baru — semua diambil verbatim dari BRD utama.
  - **Modul M7–M10 diberi link ke replit-handoff** karena sudah ada artefak implementasi yang bisa langsung dinavigasi dari file BRD per modul.
- **Results / Next Steps:**
  - Folder `document/brd/` siap digunakan sebagai referensi baca per modul.
  - Saat ada update BRD di sesi berikutnya, AI agent harus mengikuti Dual-Update Rule: update BRD utama → update file brd/<modul> terkait.

## 2026-05-09T12:00:00+07:00
- **Task Performed:** Gap analysis antara BRD v3.1 dan seluruh perubahan arsitektur yang telah dilakukan di session sebelumnya (08–09 Mei 2026). Update BRD ke versi 3.2 untuk mensinkronkan 6 area yang tidak konsisten.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — versi naik ke 3.2, 7 lokasi diupdate:
    1. Change Log: tambah entri #15–#20 yang mendokumentasikan semua perubahan arsitektur v3.2
    2. Section 2.1 M9 (Cakupan): hapus EPB Confirmation upload dari scope M9; tambah "otomatis buat row UNPAID saat APPROVED" dan "tombol Bayar EPB → navigasi ke M9b"; lifecycle tambah WAITING_PAYMENT_VERIFICATION
    3. Section 2.1 M9b (4 Status Payment): ganti "Pending Review" → "Waiting Payment Verification"; tambah catatan bahwa record UNPAID dibuat otomatis saat APPROVED; perjelas flow per status
    4. Section 3.2 End-to-End Flow Phase 1 step 6–9: revisi sesuai arsitektur baru (otomatis buat UNPAID, tombol Bayar EPB, upload di M9b, status WAITING_PAYMENT_VERIFICATION)
    5. Section 3.4.9 FR-NP-06, FR-NP-07: direvisi total + tambah FR-NP-08 (update nominations.status paralel)
    6. Section 3.4.9b FR-EI-01, FR-EI-04, FR-EI-07: update status labels + tambah FR-EI-09 (update nominations.status dalam transaksi)
    7. Appendix A.2: ganti "Pending Review" → "Waiting Payment Verification" + tambah catatan auto-create UNPAID
- **Logic / Decisions Made:**
  - **Gap 1 — Section 2.1 M9:** BRD masih menyebut customer upload proof di M9. Faktanya upload dilakukan di M9b. Fix: scope M9 diubah ke "tampilkan EPB + navigasi ke M9b".
  - **Gap 2 — Section 2.1 M9b:** Istilah "Pending Review" sudah tidak digunakan (diganti WAITING_PAYMENT_VERIFICATION). Fix: seluruh referensi diupdate.
  - **Gap 3 — Section 3.2 step 7–8:** Flow masih menggambarkan upload di M9 lalu pindah ke M9b. Fix: step 6–9 di-rewrite sesuai arsitektur baru.
  - **Gap 4 — FR-NP-06, FR-NP-07:** FR lama masih memerintahkan upload di M9. Fix: FR-NP-06 diubah jadi "tombol navigasi ke M9b"; FR-NP-07 diubah jadi "otomatis buat record UNPAID saat APPROVED"; tambah FR-NP-08 baru.
  - **Gap 5 — FR-EI-04 & FR-EI-07:** Masih pakai "Pending Review". Fix: update ke "Waiting Payment Verification" + klarifikasi webhook hanya PAYMENT_REJECT dan PAID. Tambah FR-EI-09.
  - **Gap 6 — Appendix A.2:** Status list masih "Pending Review". Fix: diupdate ke "Waiting Payment Verification".
  - **Tidak ada gap di:** Section 1 (Executive Summary), Section 3.3 (Role & Access), FR lain di 3.4.1–3.4.8 dan 3.4.10–3.4.12, Section 3.5 (NFR), Section 4 (Constraints), Section 5 (Risk), Section 6 (Acceptance Criteria), Appendix A.1 & A.3.
- **Results / Next Steps:**
  - BRD v3.2 kini sepenuhnya sinkron dengan arsitektur implementasi M9/M9b/M10.
  - Seluruh pipeline (BRD → module → replit-handoff) konsisten.
  - Pipeline siap dieksekusi ke Replit Agent: M7 → M8 → M9 → M9b → M10.

## 2026-05-09T00:00:00+07:00
- **Task Performed:** Revisi arsitektur status M9/M9b: kembalikan status `UNPAID` di M9b, tambah `WAITING_PAYMENT_VERIFICATION` di tabel `nominations`, pindah upload proof dari M9 ke M9b, buat `epb_payments UNPAID` otomatis saat webhook APPROVED.
- **Files Modified:**
  - `module/nomination-status-payment/README.md` — Key Decisions diupdate: nominations lifecycle baru, epb_payments dibuat saat APPROVED, upload di M9b
  - `module/nomination-status-payment/specifications.md` — lifecycle diagram baru, webhook APPROVED tambah INSERT epb_payments UNPAID, Section 5 diubah jadi redirect ke M9b (hapus upload flow), endpoint list diupdate
  - `module/nomination-status-payment/user-stories.md` — US-NP-01 update status labels, US-NP-02 tambah tombol Bayar EPB, US-NP-04 diubah ke "Navigate to EPB Payment Page"
  - `module/epb-invoice/README.md` — Overview diupdate, In Scope kembalikan Unpaid flow, Key Decisions diupdate (epb_payments dibuat saat APPROVED, upload di M9b, nominations.status diupdate paralel)
  - `module/epb-invoice/requirements.md` — FR-EI-01 s/d FR-EI-09: kembalikan FR-EI-03 (Unpaid Pay flow), tambah FR-EI-09 (update nominations.status paralel)
  - `module/epb-invoice/specifications.md` — lifecycle diagram dikembalikan dengan UNPAID, DB schema default = UNPAID, upload endpoint scope = UNPAID atau PAYMENT_REJECT, logic tambah update nominations.status dalam satu transaksi
  - `module/epb-invoice/user-stories.md` — kembalikan US-EI-03 (Pay Unpaid EPB), US-EI-04 (Monitor Waiting), renumber US-EI-05 s/d US-EI-06
  - `module/customer-dashboard-monitoring/README.md` — Key Decision diupdate: nominations.status sekarang juga WAITING_PAYMENT_VERIFICATION
  - `module/customer-dashboard-monitoring/specifications.md` — status list tambah WAITING_PAYMENT_VERIFICATION, Active filter note diupdate
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — Step 1 status list update, Step 3 webhook APPROVED tambah INSERT epb_payments UNPAID, Step 6 diubah jadi "Tombol Bayar EPB" (hapus upload backend), Step 7 status banner tambah WAITING_PAYMENT_VERIFICATION, acceptance checklist diupdate
  - `implementation/replit-handoff/m9b-epb-invoice.md` — Prerequisites update, context update, Step 1 DB schema default UNPAID, Step 2 model default UNPAID, Step 5 kembalikan untuk UNPAID+PAYMENT_REJECT + tambah update nominations.status dalam transaksi, Step 7 kembalikan "Pay" button, Step 8 kembalikan Pay action section + upload modal shared, acceptance checklist diupdate
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md` — statusLabels map tambah WAITING_PAYMENT_VERIFICATION, status badge list tambah WAITING_PAYMENT_VERIFICATION (yellow)
- **Logic / Decisions Made:**
  - **Arsitektur baru:** `epb_payments` tetap punya `UNPAID` sebagai status awal — dibuat otomatis oleh M9 webhook handler saat event APPROVED diterima, tanpa aksi customer. Ini analog dengan `nominations.status = APPROVED` yang juga dibuat tanpa aksi customer.
  - **Upload proof sepenuhnya di M9b:** M9 tidak punya endpoint upload. M9 hanya menampilkan EPB detail dan tombol "Bayar EPB" yang mengarah ke M9b. Ini konsisten dengan arsitektur M9b sebagai payment management module.
  - **nominations.status dan epb_payments.status bergerak paralel:** Setelah customer upload proof di M9b, M9b backend mengupdate kedua tabel dalam satu transaksi DB. Ini memungkinkan M10 menampilkan status `WAITING_PAYMENT_VERIFICATION` di daftar nominasi tanpa perlu join ke `epb_payments`.
  - **Satu upload endpoint untuk dua flow:** `POST /api/customer/epb-payments/:id/proof` melayani UNPAID (Pay) dan PAYMENT_REJECT (Revision Data), dibedakan oleh current status saja.
- **Results / Next Steps:**
  - Semua 4 layer pipeline (module → replit-handoff) untuk M9, M9b, M10 kini sinkron dengan arsitektur baru.
  - Urutan implementasi: M7 → M8 → M9 → M9b → M10.

## 2026-05-08T00:00:00+07:00
- **Task Performed:** Hapus status `EPB_CONFIRMATION_SUBMITTED` dari M9 dan status `UNPAID` dari M9b. Menyederhanakan flow payment: setelah customer submit EPB Confirmation di M9, status nominasi tetap `APPROVED` dan `epb_payments` row langsung dibuat dengan status `WAITING_PAYMENT_VERIFICATION`.
- **Files Modified:**
  - `module/nomination-status-payment/README.md` — Key Decisions diupdate: hapus EPB_CONFIRMATION_SUBMITTED, tambah penjelasan nominations tetap APPROVED, epb_payments langsung WAITING_PAYMENT_VERIFICATION
  - `module/nomination-status-payment/specifications.md` — lifecycle diagram diupdate, status list direvisi, upload & submit flow Step 3-4 diubah, note tentang tidak ada EPB_CONFIRMATION_SUBMITTED/UNPAID ditambahkan
  - `module/nomination-status-payment/user-stories.md` — US-NP-01 tambah AC untuk APPROVED setelah proof submitted, US-NP-02 revisi, US-NP-04 acceptance criteria diupdate seluruhnya
  - `module/epb-invoice/README.md` — Overview diupdate, In Scope direvisi (hapus Unpaid flow), Key Decisions diupdate (hapus UNPAID, hapus EPB_PENDING_REVIEW webhook, perjelas upload endpoint hanya untuk PAYMENT_REJECT), Dependencies diupdate
  - `module/epb-invoice/requirements.md` — FR-EI-01 s/d FR-EI-08 direvisi: hapus FR-EI-03 (Unpaid Pay flow), renumber, update status labels ke WAITING_PAYMENT_VERIFICATION
  - `module/epb-invoice/specifications.md` — lifecycle diagram diubah total (hapus UNPAID), DB schema default diubah ke WAITING_PAYMENT_VERIFICATION, webhook hapus EPB_PENDING_REVIEW, upload endpoint scope diubah ke PAYMENT_REJECT only, status di retry comment diupdate
  - `module/epb-invoice/user-stories.md` — US-EI-03 diganti (Pay flow → Waiting Verification view-only), US-EI-04 s/d US-EI-06 direnumber ke US-EI-04 s/d US-EI-05
  - `module/customer-dashboard-monitoring/README.md` — Key Decision status scope diupdate: hapus EPB_CONFIRMATION_SUBMITTED, tambah klarifikasi APPROVED tetap setelah submit
  - `module/customer-dashboard-monitoring/specifications.md` — status list di §2 hapus EPB_CONFIRMATION_SUBMITTED, "Active" filter note diupdate
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — Step 1 status list diupdate, Step 6 logic diubah total (hapus EPB_CONFIRMATION_SUBMITTED, tambah WAITING_PAYMENT_VERIFICATION, tambah epb_payment_proofs insert, tambah note jangan ubah nominations.status), Step 7 status banner table diupdate, acceptance checklist diupdate
  - `implementation/replit-handoff/m9b-epb-invoice.md` — Prerequisites diupdate, Step 1 DB schema default diubah, Step 2 model default diubah, Step 3 hapus EPB_PENDING_REVIEW handler + tambah note, Step 5 judul + validasi diubah (PAYMENT_REJECT only), Step 6 retry note diupdate, Step 7 layout hapus "Pay" button + update status badges, Step 8 tambah Waiting Verification info box + hapus Pay action, upload modal diubah jadi Revision Data only, acceptance checklist diupdate
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md` — query exclude list hapus EPB_CONFIRMATION_SUBMITTED, statusLabels map hapus EPB_CONFIRMATION_SUBMITTED, filter active note diupdate, status badge list hapus EPB_CONFIRMATION_SUBMITTED
- **Logic / Decisions Made:**
  - **Keputusan utama:** `EPB_CONFIRMATION_SUBMITTED` dan `UNPAID` adalah dua nama berbeda untuk state yang sama (customer sudah upload proof, menunggu verifikasi Finance). Digabung menjadi satu status: `WAITING_PAYMENT_VERIFICATION` yang langsung di-set ketika M9 Step 6 dijalankan.
  - **Status nominasi tidak berubah:** Setelah EPB Confirmation submit, `nominations.status` tetap `APPROVED`. Tracking payment sepenuhnya delegasi ke `epb_payments`.
  - **M9b tidak punya "initial upload" flow:** Tombol "Pay" di M9b dihapus. Entry point selalu dari M9. M9b hanya mengelola re-upload ketika `PAYMENT_REJECT`.
  - **EPB_PENDING_REVIEW webhook dihapus:** Tidak dibutuhkan karena LPS sendiri yang men-set `WAITING_PAYMENT_VERIFICATION` saat proof di-upload — tidak perlu konfirmasi dari STS.
  - **Idempotency guard M9 diubah:** Bukan cek `status = EPB_CONFIRMATION_SUBMITTED` lagi, melainkan cek apakah row `epb_payments` sudah ada untuk nomination EPB tersebut.
- **Results / Next Steps:**
  - Semua 4 layer pipeline (module → replit-handoff) untuk M9, M9b, dan M10 kini sinkron dengan keputusan baru.
  - Flow customer lebih sederhana: Upload proof di M9 → langsung masuk WAITING_PAYMENT_VERIFICATION di M9b → tunggu STS → selesai (PAID) atau upload ulang (PAYMENT_REJECT).
  - Pipeline siap dihandoff ke Replit Agent: M7 → M8 → M9 → M9b → M10.

## 2026-05-07T01:00:00+07:00
- **Task Performed:** Analisis dampak perubahan M9/M9b terhadap M10 dan sinkronisasi yang diperlukan.
- **Files Modified:**
  - `module/customer-dashboard-monitoring/README.md` — tambah Key Decision baru: M10 hanya baca tabel `nominations`; status payment ada di `epb_payments` (M9b); nominasi `EPB_CONFIRMATION_SUBMITTED` menampilkan shortcut ke M9b
  - `module/customer-dashboard-monitoring/specifications.md` — dua update: (1) status list di §2 diganti `WAITING_PAYMENT_VERIFICATION`/`PAYMENT_REJECTED` → `EPB_CONFIRMATION_SUBMITTED` + catatan penjelasan; (2) "Active filter" ditambah catatan excludes `EPB_CONFIRMATION_SUBMITTED`
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md` — empat update: (1) query `active_nominations` tambah `EPB_CONFIRMATION_SUBMITTED` di exclude list; (2) `statusLabels` map hapus `WAITING_PAYMENT_VERIFICATION`/`PAYMENT_REJECTED`, tambah `EPB_CONFIRMATION_SUBMITTED`; (3) filter `?filter=active` exclude `EPB_CONFIRMATION_SUBMITTED`; (4) status badge colors hapus `WAITING_PAYMENT_VERIFICATION`/`PAYMENT_REJECTED`, ganti dengan `EPB_CONFIRMATION_SUBMITTED` (teal + link ke M9b); (5) sidebar nav tambah "EPB & Invoice → /customer/epb-invoice"
- **Logic / Decisions Made:**
  - M10 requirements (FR-CD-01 s/d FR-CD-08) dan user-stories tidak berubah — semuanya masih valid.
  - Perubahan bersifat teknis: menyesuaikan referensi status lama M9 dengan status baru setelah pemisahan M9/M9b.
  - `EPB_CONFIRMATION_SUBMITTED` dikecualikan dari filter "Active" karena dari perspektif M10, aksi customer sudah berpindah ke M9b. Nominasi ini tetap tampil di tab "All" agar customer masih bisa melihat riwayat.
  - Sidebar nav M10 ditambah "EPB & Invoice" sebagai navigasi global ke M9b — konsisten dengan M9b handoff Step 9.
- **Results / Next Steps:**
  - Semua 4 layer M10 kini sinkron dengan BRD v3.1.
  - Seluruh pipeline M7 → M8 → M9 → M9b → M10 sudah konsisten dan siap dihandoff ke Replit Agent secara sequential.
  - Tidak ada modul lain yang terdampak — M8, M11, M12 tidak memiliki referensi ke status payment M9 lama.

## 2026-05-07T00:00:00+07:00
- **Task Performed:** Sinkronisasi penuh M9 dan pembuatan M9b berdasarkan perubahan BRD v3.1 (Swimlane V3). Meliputi update 4 layer pipeline implementasi untuk M9, dan pembuatan dari nol untuk M9b.
- **Files Modified:**
  - `module/nomination-status-payment/README.md` — judul diubah ke "Nomination Status & EPB Confirmation", boundaries direvisi: scope baru hanya sampai EPB Confirmation submit; out-of-scope ditambah payment verification cycle (M9b)
  - `module/nomination-status-payment/requirements.md` — FR-NP-08 s/d FR-NP-11 dihapus (dipindah ke M9b sebagai FR-EI-01 s/d FR-EI-08); FR-NP-07 direvisi: setelah Submit EPB Confirmation, data dipindahkan ke menu EPB & Invoice (M9b)
  - `module/nomination-status-payment/specifications.md` — rewrite penuh: status lifecycle dipersingkat (tidak ada WAITING_PAYMENT_VERIFICATION/PAYMENT_CONFIRMED/PAYMENT_REJECTED); Step 5 (EPB Confirmation flow) direvisi: setelah submit membuat row `epb_payments` di M9b dan set status `EPB_CONFIRMATION_SUBMITTED`; STS webhook dibatasi hanya APPROVED dan NEED_REVISION
  - `module/nomination-status-payment/user-stories.md` — US-NP-04 dan US-NP-05 (payment verification) dihapus; US-NP-04 baru dibuat: "Submit EPB Confirmation (First Payment Proof)" dengan AC yang benar
  - `implementation/replit-handoff/m9-nomination-status-payment.md` — rewrite penuh ke v2.0: Step 6 diubah dari "Payment Proof Upload → WAITING_PAYMENT_VERIFICATION" menjadi "EPB Confirmation Submit → create epb_payments(UNPAID) + redirect ke M9b"; status `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_CONFIRMED`, `PAYMENT_REJECTED` dihapus; webhook handler dibatasi APPROVED + NEED_REVISION; acceptance checklist diupdate
  - `module/epb-invoice/README.md` — **file baru**: overview M9b, boundaries, dependencies, key decisions
  - `module/epb-invoice/requirements.md` — **file baru**: FR-EI-01 s/d FR-EI-08 dari BRD §3.4.9b
  - `module/epb-invoice/specifications.md` — **file baru**: payment lifecycle (UNPAID→PENDING_REVIEW→PAYMENT_REJECT/PAID), DB schema (epb_payments + epb_payment_proofs), STS webhook (EPB_PENDING_REVIEW, EPB_PAYMENT_REJECT, EPB_PAID), upload endpoint, list/detail API, frontend routes
  - `module/epb-invoice/user-stories.md` — **file baru**: US-EI-01 s/d US-EI-06 dengan acceptance criteria lengkap
  - `implementation/replit-handoff/m9b-epb-invoice.md` — **file baru**: full Replit Agent handoff (9 steps + acceptance checklist)
- **Logic / Decisions Made:**
  - **Pemisahan M9 / M9b sesuai Swimlane V3:** M9 berakhir tepat setelah customer submit EPB Confirmation (first proof upload). Status `WAITING_PAYMENT_VERIFICATION` dihapus dari M9 — diganti dengan status terminal `EPB_CONFIRMATION_SUBMITTED` di M9. Siklus verifikasi (UNPAID → PENDING_REVIEW → PAYMENT_REJECT → PAID) sepenuhnya di M9b.
  - **Kepemilikan tabel `epb_payments`:** Tabel dibuat oleh migrasi M9b, tetapi INSERT row pertama dilakukan oleh M9 Step 6 (EPB Confirmation Submit). Ini memungkinkan M9 dan M9b dikerjakan secara sequential tanpa circular dependency.
  - **Webhook separation:** M9 webhook handler (`POST /api/webhooks/sts/nomination-status`) hanya menangani APPROVED dan NEED_REVISION. M9b punya webhook handler terpisah (`POST /api/webhooks/sts/epb-payment-status`) untuk EPB_PENDING_REVIEW, EPB_PAYMENT_REJECT, EPB_PAID.
  - **Upload endpoint M9b:** Satu endpoint `POST /api/customer/epb-payments/:id/proof` melayani dua flow (Pay dari UNPAID, dan Revision Data dari PAYMENT_REJECT) — dibedakan oleh current status, bukan mode flag.
  - **M8 tidak terdampak:** Requirements, specifications, user-stories, dan replit-handoff M8 tidak perlu diubah.
- **Results / Next Steps:**
  - Semua 4 layer pipeline (module → replit-handoff) kini sinkron dengan BRD v3.1 untuk M9 dan M9b.
  - M9b siap dihandoff ke Replit Agent menggunakan `implementation/replit-handoff/m9b-epb-invoice.md` setelah M9 selesai.
  - Urutan implementasi yang disarankan: M7 → M8 → M9 → M9b → M10.
  - Tidak ada perubahan yang diperlukan di M8, M10, atau modul lainnya.

## 2026-05-06T13:00:00+07:00
- **Task Performed:** Gap analysis Swimlane V3 vs BRD V3, lalu update BRD untuk sinkronisasi. BRD dinaikkan ke v3.1.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md` — versi naik ke 3.1, 6 lokasi diupdate
- **Logic / Decisions Made:**
  - **Gap 1 — EPB & Invoice menu tidak ada di BRD:** Swimlane terbaru mengkonfirmasi menu ini tetap ada di Customer Portal LPS dengan 4 status payment. BRD sebelumnya merekomendasikan dihapus. Fix: ditambahkan sebagai **M9b** dengan scope dan 8 FR baru (FR-EI-01 s/d FR-EI-08).
  - **Gap 2 — EPB Confirmation flow di M9 terlalu kompleks:** BRD M9 menggabungkan EPB Confirmation + siklus verifikasi payment dalam satu modul. Swimlane terbaru memisahkan keduanya: M9 hanya sampai "Submit → data pindah ke EPB & Invoice menu", siklus Unpaid/Pending/Reject/Paid ada di M9b. Fix: M9 direvisi namanya menjadi "Nomination Status & EPB Confirmation", scope dan FR dipersempit; FR-NP-08 s/d FR-NP-11 (yang dulu) dipindahkan ke FR-EI-* di section baru.
  - **Gap 3 — Tabel modul tidak mencantumkan M9b:** Fix: ditambahkan di tabel section 3.1.
  - **Gap 4 — Tabel hubungan LPS-STS di section 1.3 tidak akurat:** Billing row terlalu simpel. Fix: dipisah menjadi 2 baris — STS mengelola kalkulasi/invoice/reconciliation; LPS mengelola EPB & Invoice interface (Customer Portal).
  - **Gap 5 — Out of Scope item 8 ambigu:** Bisa salah dibaca bahwa LPS tidak boleh punya apapun terkait billing. Fix: ditambahkan catatan klarifikasi.
  - **Gap 6 — Appendix A.2 tidak konsisten:** Fix: ditambahkan sub-section "Yang tetap di LPS" untuk EPB & Invoice Customer Portal.
  - **Extra Features (Customer Approval + Add Customer By Admin):** sudah ada di BRD (FR-CA-06 s/d FR-CA-09) — tidak ada gap.
  - **Operator Swimlane:** tidak ada gap dengan BRD.
- **Results / Next Steps:**
  - BRD v3.1 kini sinkron dengan Swimlane V3 terbaru.
  - Module yang perlu perhatian di layer implementasi:
    - M9 `module/nomination-status-payment/` perlu diupdate (scope baru: sampai EPB Confirmation saja)
    - M9b belum punya `module/` folder — perlu dibuat, atau digabung ke M9 dengan sub-section
    - `implementation/replit-handoff/m9-nomination-status-payment.md` perlu direvisi untuk mencerminkan pemisahan ini

## 2026-05-06T12:00:00+07:00
- **Task Performed:** Update Swimlane Analysis LPS V3 berdasarkan screenshot Swimlane terbaru dari tim.
- **Files Modified:**
  - `document/Swimlane-Analysis-LPS V3.md`
- **Logic / Decisions Made:**
  - **EPB & Invoice DIPERTAHANKAN** (koreksi dari analisis sebelumnya yang merekomendasikan hapus): Screenshot mengkonfirmasi proses ini tetap ada di Customer Portal LPS dengan 4 status payment — Unpaid (Click Pay → Upload → Submit), Pending Review, Payment Reject (Click Revision Data → Re-upload), Paid.
  - **EPB Confirmation flow disederhanakan**: Flow terbaru hanya 2 branch — True (Approved: view EPB detail → Upload Proof of Payment → Submit → data pindah ke EPB & Invoice menu) dan False (Update Nomination Data (REVISION) → Submit Nomination Update). Branch "Need Revision" terpisah dihapus, disatukan dalam branch False.
  - **Extra Features baru ditambahkan** (bagian bawah screenshot berlabel "Extra Feature"):
    1. **Customer Approval** (Admin): Login → Customer Management → View Detail Customer Registration → Approve/Reject → Send notif to customer
    2. **Add Customer By Admin**: Login → Customer Management → Add new customer → Fill Customer data → (Optional: add custom document) → Save
  - Extra Features ini sudah terimplementasi di M7 v1.1 (replit-handoff), sehingga swimlane kini sinkron dengan dokumentasi implementasi.
  - Section Rekomendasi diupdate: tabel prioritas direvisi, item "Hapus EPB & Invoice" diganti dengan "Pertahankan EPB & Invoice dengan 4 status", tambah status "✅ Sudah di M7 v1.1" untuk Customer Approval.
- **Results / Next Steps:**
  - Swimlane Analysis V3 kini sinkron dengan screenshot terbaru dari tim.
  - Perlu dipertimbangkan apakah M9 (Nomination Status & Payment) scope-nya perlu diupdate untuk mencerminkan: (1) EPB Confirmation flow 2-branch sederhana, (2) data payment masuk ke EPB & Invoice menu (Proses 6).
  - EPB & Invoice sebagai menu terpisah mungkin perlu modul atau sub-modul tersendiri, atau digabung ke M9.

## 2026-05-06T11:00:00+07:00
- **Task Performed:** Update replit-handoff M7 ke v1.1 (versioning) dan buat file delta baru untuk perubahan Admin Customer Management & Add Customer.
- **Files Modified:**
  - `implementation/replit-handoff/m7-customer-authentication.md` — update ke v1.1 dengan version header, DB schema baru, model baru, repository baru, admin handler baru (approve, reject+email, list all, add manual), frontend Step 14 refactor + Step 15 baru, API summary update, acceptance checklist dipisah v1.0 / v1.1
  - `implementation/replit-handoff/m7-customer-authentication-v1.1-delta.md` — **file baru**, hanya memuat delta perubahan v1.1 (Step A–H: 2 alter migration, model update, repo update, email service, handler update, 3 frontend pages, router update, env vars, acceptance checklist v1.1)
- **Logic / Decisions Made:**
  - File existing diupdate dengan versioning di header (v1.0 → v1.1) dan tanda `⚠️ v1.1 change` di setiap section yang berubah — agar mudah dibaca diff-nya.
  - File delta (v1.1-delta) berdiri sendiri: bisa dijalankan ke Replit Agent secara independen jika v1.0 sudah selesai. Step diberi huruf (A–H) agar tidak bentrok dengan step di file induk.
  - Endpoint rename: `/activate` → `/approve` (lebih sesuai bahasa UI flow di screenshot).
  - Email notification: 3 template — approval, rejection, welcome (admin-add). Dikonfigurasi via SMTP env vars + FRONTEND_URL.
  - Frontend: `PendingCustomersPage` direfactor menjadi `CustomerManagementPage` (full list + filter + detail page); `AddCustomerPage` dibuat baru dengan `useFieldArray` untuk dynamic custom documents.
- **Results / Next Steps:**
  - Semua layer dokumentasi (BRD → module → replit-handoff) sudah konsisten untuk fitur Admin Approval dan Admin Add Customer.
  - Siap dihandoff ke Replit Agent: gunakan `m7-customer-authentication-v1.1-delta.md` jika v1.0 sudah live, atau `m7-customer-authentication.md` untuk fresh implementation.

## 2026-05-06T10:00:00+07:00
- **Task Performed:** Sinkronisasi 2 fitur baru Admin ke semua layer dokumentasi M7: (1) Customer Approval flow, (2) Admin Add Customer manual dengan custom documents.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
  - `module/customer-authentication/requirements.md`
  - `module/customer-authentication/specifications.md`
  - `module/customer-authentication/user-stories.md`
- **Logic / Decisions Made:**
  - FR-CA-04 s/d FR-CA-05 lama diganti/diperluas menjadi FR-CA-04 s/d FR-CA-10 untuk memisahkan concern secara eksplisit: list customer (04), view detail (05), approve (06), reject (07), admin add (08), custom docs (09), customer login (10).
  - Customer Approval: endpoint `PUT /api/admin/customers/:id/approve` (bukan `/activate`) untuk konsistensi bahasa dengan UI flow.
  - Notifikasi email dikirim pada kedua skenario approve dan reject — ini adalah kebutuhan eksplisit dari screenshot flow yang diberikan.
  - Admin Add Customer: akun langsung ACTIVE, Customer Code langsung di-generate, tidak ada approval step.
  - Custom documents: ditandai `is_custom = true` di tabel `customer_documents`. Kolom `doc_label` digunakan untuk nama bebas yang diisi Admin. Kolom `registration_source` di tabel `customers` ditambahkan (`SELF` / `ADMIN`) untuk membedakan asal pembuatan akun.
  - User stories direfactor: US-CA-02 menjadi "View Customer Management", US-CA-03 Approve, US-CA-04 Reject, US-CA-05 Admin Add Manual, US-CA-06 Customer Login (renumber dari US-CA-03 lama).
- **Results / Next Steps:**
  - BRD, requirements, specifications, user-stories sudah sinkron untuk fitur Admin Approval dan Admin Add Customer.
  - Layer berikutnya yang perlu diupdate: `implementation/replit-handoff/m7-customer-authentication.md` dan `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`.

## 2026-05-06T04:00:00+07:00
- **Task Performed:** Sinkronisasi perubahan routing dari Replit ke semua layer dokumentasi.
- **Files Modified:**
  - `implementation/replit-handoff/m7-customer-authentication.md`
  - `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md`
  - `module/customer-authentication/specifications.md`
- **Logic / Decisions Made:**
  - Customer login route: `/login` → `/customer/login` di semua file.
  - Token key: `lps_token` → `customer_token` di localStorage (customer auth hanya pakai localStorage, bukan cookie).
  - Admin auth: endpoint `POST /api/auth/login`, token disimpan di HTTP-only cookie, route login admin `/admin/login` — sepenuhnya terpisah dari customer auth.
  - Admin protected routes semua di bawah prefix `/admin/` (e.g., `/admin/customers/pending`).
  - JWT middleware customer: hanya baca `Authorization: Bearer` header (hapus referensi ke cookie `lps_token`).
  - Logout handler: `localStorage.removeItem('customer_token')` + redirect ke `/customer/login`.
  - Default root redirect: `/` → `/customer/login` (bukan `/login`).
  - PendingCustomersPage: tambah note route `/admin/customers/pending` + redirect ke `/admin/login` jika tidak authenticated.
  - m9: tidak ada referensi `/login` — tidak perlu diubah.
- **Results / Next Steps:**
  - Semua layer (replit-handoff + module specs) sudah konsisten dengan route structure aktual di Replit.
  - M7–M10 siap dihandoff ulang ke Replit jika diperlukan update.

## 2026-05-06T03:00:00+07:00
- **Task Performed:** Hapus field Tipe Pelanggan dari form registrasi customer; sistem default ke "Cargo Owner".
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
  - `module/customer-authentication/requirements.md`
  - `module/customer-authentication/specifications.md`
  - `module/customer-authentication/user-stories.md`
  - `implementation/design/2026-04-25-module7-decomposition-design.md`
  - `implementation/replit-handoff/m7-customer-authentication.md`
  - `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
  - `implementation/replit-handoff/m8-nomination-submission.md`
- **Logic / Decisions Made:**
  - Field `type` diubah dari `TEXT[]` (multi-select) menjadi `VARCHAR(50) DEFAULT 'Cargo Owner'` di semua DB schema.
  - Go model diubah dari `pq.StringArray` menjadi `string` (tidak perlu import lib/pq).
  - Form registrasi: checkbox Type dihapus; handler tidak menerima `type[]` dari request — selalu hardcode `type = "Cargo Owner"` saat insert.
  - STS sync payload: `"type"` berubah dari array ke string `"Cargo Owner"`.
  - mockCustomer di prototype diupdate dari array ke string.
- **Results / Next Steps:**
  - Semua layer sudah konsisten — field Type tidak muncul di form manapun.

## 2026-05-06T02:00:00+07:00
- **Task Performed:** Sinkronisasi fitur Document Master ke semua layer dokumentasi. Fitur ini sebelumnya sudah diprompt ke Replit namun belum tersinkron ke repo ini.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
  - `module/customer-dashboard-monitoring/requirements.md`
  - `module/customer-dashboard-monitoring/README.md`
  - `module/customer-dashboard-monitoring/specifications.md`
  - `module/customer-dashboard-monitoring/user-stories.md`
  - `module/nomination-submission/specifications.md`
  - `implementation/replit-handoff/m8-nomination-submission.md`
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md`
  - `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
- **Logic / Decisions Made:**
  - Document Master ditempatkan di M10 (bukan modul baru) karena merupakan fitur customer portal yang bersifat supporting — tidak punya business flow sendiri, hanya repositori dokumen.
  - Prinsip utama: **append-only** — API DELETE selalu return 403 Forbidden, tidak ada tombol delete/edit di UI.
  - Dokumen saat "Upload Baru" di form nominasi **otomatis tersimpan ke Document Master** (satu tulis, dua manfaat).
  - `nomination_documents` ditambah FK `document_master_id` (nullable): NULL jika upload langsung, terisi jika dipilih dari Document Master — referensi by ID, bukan duplikasi file.
  - DB: tabel `customer_document_master` tanpa `updated_at`/`deleted_at` untuk menegaskan sifat immutable secara skema.
  - BRD: FR-CD-05 sampai FR-CD-08 ditambahkan di section 3.4.10.
  - M8 replit-handoff: endpoint `POST /api/customer/nominations/:id/documents` diupdate support 2 mode (multipart upload vs JSON select from master).
  - M10 replit-handoff: tambah Step 9 (backend Document Master endpoints) dan Step 10 (frontend DocumentMasterPage), Step 11 navigation update.
  - Prototype: tambah mockDocuments, DocumentMasterPage, route `/customer/documents`, dual upload tab di NominationFormPage, nav item FolderOpen, demo flow & checklist diupdate.
- **Results / Next Steps:**
  - Semua 4 layer sudah sinkron: BRD → module → design (tidak perlu update karena design doc hanya scope M7) → replit-handoff.
  - Siap dihandoff ke Replit untuk implementasi Document Master di M10 dan update M8 form.

## 2026-05-06T01:00:00+07:00
- **Task Performed:** Update semua file implementasi M7 menyesuaikan perubahan form registrasi customer.
- **Files Modified:**
  - `implementation/design/2026-04-25-module7-decomposition-design.md`
  - `implementation/replit-handoff/m7-customer-authentication.md`
  - `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
- **Logic / Decisions Made:**
  - Design doc: scope M7 dan FR ID mapping diupdate ke FR-CA-01 s/d FR-CA-05.
  - Full-stack handoff (m7-customer-authentication.md): total rewrite mencakup:
    - Migration baru: tabel `customers` (kolom baru: customer_code, type[], pic_name, address, note) + tabel `customer_documents` (NPWP/NIP/Company Profile)
    - Model Go diupdate (`pq.StringArray` untuk type[], relasi ke `CustomerDocument`)
    - Handler registrasi diubah ke `multipart/form-data` dengan 3 file upload wajib
    - Admin handler ditambah: `GET /api/admin/customers/:id` (detail + dokumen), logika generate `customer_code` format `CUST-YYYYMM-XXXXX`
    - STS sync payload diupdate (customer_code, type, pic_name, address)
    - Frontend RegisterPage: 2 section (Company Information + Required Documents dengan 3 document card)
    - Acceptance checklist diupdate lengkap
  - UI prototype handoff: Register Page diupdate sesuai screenshot; mockCustomer field diupdate (customerCode, customerName, type[], picName); demo flow ditambah langkah register; checklist ditambah
- **Results / Next Steps:**
  - Semua 4 layer dokumentasi (BRD → module → design → replit-handoff) sudah konsisten.
  - Siap dihandoff ke Replit Agent untuk implementasi M7.
  - Konfirmasi masih terbuka: apakah password dibuat saat registrasi atau dikirim via email setelah aktivasi?

## 2026-05-06T00:30:00+07:00
- **Task Performed:** Update BRD-LPS-V3-Analysis.md untuk menyinkronkan perubahan form registrasi M7.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
- **Logic / Decisions Made:**
  - Section 2.1 (cakupan modul 7) diupdate: tambah field baru (Type, PIC Name, Address, Note), dokumen wajib (NPWP/NIP/Company Profile file upload), dan Customer Code auto-generate.
  - Section 3.4.7 (FR-CA) diupdate: FR-CA-01 direvisi dengan field lengkap; tambah FR-CA-02 (dokumen wajib), FR-CA-03 (Customer Code auto-generate), FR-CA-04 (Admin validasi + review dokumen); FR sebelumnya digeser nomornya.
- **Results / Next Steps:**
  - BRD, module/, dan screenshots sudah sinkron.
  - Konfirmasi masih diperlukan: apakah password dibuat saat registrasi atau dikirim via email setelah aktivasi?

## 2026-05-06T00:00:00+07:00
- **Task Performed:** Sinkronisasi data registrasi customer M7 berdasarkan screenshot UI aktual dari tim pengembang.
- **Files Modified:**
  - `module/customer-authentication/requirements.md`
  - `module/customer-authentication/specifications.md`
  - `module/customer-authentication/user-stories.md`
- **Logic / Decisions Made:**
  - Form registrasi aktual berbeda signifikan dari spesifikasi awal. Perubahan utama:
    1. **Field baru:** Customer Code (auto-generated), Type (multi-checkbox: Cargo Owner, Shipper, PBM, Agen, Surveyor, Vendor Dozer), PIC Name, Address, Note
    2. **Field diubah:** Contact Person → PIC Name
    3. **Dokumen wajib ditambahkan:** Upload 3 dokumen (NPWP file, NIP file, Company Profile file) masing-masing dengan Description, Issue Date, Expiry Date
    4. **Customer Code** tidak ditampilkan di form (read-only, auto-generated saat Admin approve)
    5. **Password** tidak ada di form registrasi (kemungkinan dikirim via email setelah aktivasi, atau flow terpisah — perlu konfirmasi)
  - Database schema diupdate: tambah kolom `customer_code`, `type[]`, `pic_name`, `address`, `note`; tambah tabel `customer_documents`
  - STS sync payload diupdate menyertakan field-field baru
- **Results / Next Steps:**
  - Konfirmasi dengan tim: apakah password dibuat sendiri saat registrasi atau dikirim via email setelah aktivasi?
  - Jika ada replit-handoff yang sudah dibuat untuk M7, perlu diupdate menyesuaikan perubahan form registrasi ini.

## 2026-04-25T23:30:00+07:00
- **Task Performed:** Created a frontend-only UI prototype replit-handoff covering all 4 customer portal modules (M7–M10) as a single deliverable for client presentation.
- **Files Modified:**
  - `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
- **Logic / Decisions Made:**
  - No backend, no database, no API calls — pure React frontend with mock data.
  - All mock data is realistic (Indonesian vessel names, Rupiah amounts, WITA timezone, Bahasa Indonesia labels).
  - React useState used for in-page status transitions (e.g., payment upload → WAITING_PAYMENT_VERIFICATION) to make the demo feel interactive.
  - Includes a guided demo flow for the client presentation walkthrough.
  - Separate from the full-implementation handoffs (m7–m10 individual files) which remain intact for future actual development.
- **Results / Next Steps:**
  - Deliver `m7-m10-customer-portal-ui-prototype.md` to Replit Agent for UI prototype build.
  - Use the individual m7–m10 handoffs later for full backend + database implementation.

## 2026-04-25T23:00:00+07:00
- **Task Performed:** Created all module/ documentation (16 files) and replit-handoff prompts (4 files) for M7–M10 Customer Portal modules.
- **Files Modified:**
  - `module/customer-authentication/` (README, requirements, specifications, user-stories)
  - `module/nomination-submission/` (README, requirements, specifications, user-stories)
  - `module/nomination-status-payment/` (README, requirements, specifications, user-stories)
  - `module/customer-dashboard-monitoring/` (README, requirements, specifications, user-stories)
  - `implementation/replit-handoff/m7-customer-authentication.md`
  - `implementation/replit-handoff/m8-nomination-submission.md`
  - `implementation/replit-handoff/m9-nomination-status-payment.md`
  - `implementation/replit-handoff/m10-customer-dashboard-monitoring.md`
  - `implementation/plan/customer-portal-plan.md`
- **Logic / Decisions Made:**
  - Tech stack: React 19 + Vite + TailwindCSS v4 + shadcn/ui + Recharts + Framer Motion + wouter + TanStack React Query (frontend); Go 1.25 + jasoet/pkg/v2 (backend); PostgreSQL + golang-migrate.
  - Module files derived directly from BRD (no speculation).
  - Replit handoff files include: DB migrations (SQL), Go backend handlers/models, frontend React pages, and acceptance checklists. Each is self-contained.
  - M7 handoff covers: registration, admin validation, login, STS sync, JWT middleware.
  - M8 handoff covers: nomination form, draft, document upload, submit, STS API submission.
  - M9 handoff covers: STS webhook (HMAC verified), status tracking, EPB view, payment proof upload, revision flow.
  - M10 handoff covers: dashboard aggregation API, nomination list, voyage tracking (Leaflet map + AIS), weather widget.
- **Results / Next Steps:**
  - All 4 modules are ready to be handed off to the Replit Agent, starting from M7.
  - Deliver handoffs sequentially: M7 → M8 → M9 → M10 (each depends on previous).

## 2026-04-25T22:30:00+07:00
- **Task Performed:** Decomposed BRD Module 7 (Nomination Request / Customer Portal) into 4 focused modules (M7–M10). Renumbered former M8→M11 and M9→M12. Updated all affected BRD sections.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
  - `implementation/design/2026-04-25-module7-decomposition-design.md` (design artifact)
- **Logic / Decisions Made:**
  - Split driven by swimlane 1:1 mapping: each customer-side swimlane process becomes one module.
  - New modules: M7 Customer Authentication & Onboarding, M8 Nomination Request Submission, M9 Nomination Status & Payment, M10 Customer Dashboard & Monitoring.
  - FR IDs renamed: FR-NR-* retired; replaced by FR-CA-* (M7), FR-NS-* (M8), FR-NP-* (M9), FR-CD-* (M10).
  - Module count grows from 9 to 12. Sections 2.1, 3.1, 3.3, and 3.4 updated accordingly.
  - module/ folders and implementation/ artifacts not created yet — deferred to next task.
- **Results / Next Steps:**
  - BRD is now fully decomposed with clear module boundaries.
  - Next: create module/<sub-module>/ folders (4-file structure) for M7–M10 and populate from BRD.

## 2026-04-25T22:00:00+07:00
- **Task Performed:** Synced BRD (Module 7 — Nomination Request) with Swimlane Analysis V3 Customer Side. Closed 3 gaps identified by cross-referencing both documents.
- **Files Modified:**
  - `document/BRD-LPS-V3-Analysis.md`
- **Logic / Decisions Made:**
  - Gap 1: FR-NR-09 only listed "Approved, Need Revision" — added "Pending" with display label "Menunggu proses di STS Platform" to match swimlane Process 3 explicit branch.
  - Gap 2: Customer Portal Dashboard had no scope or FR — added scope subsection under Module 7 and FR-NR-22.
  - Gap 3: Customer weather access was in the access matrix but had no FR — added FR-NR-23 and dashboard scope bullet. Also updated FR-NR-21 to explicitly include Draft nominations in the Nomination Status list.
  - Status lifecycle in Tracking & Monitoring scope updated to full sequence: Draft → Submitted → Pending → Approved/Need Revision → Waiting Payment Verification → Payment Confirmed/Payment Rejected.
- **Results / Next Steps:**
  - BRD Module 7 now fully reflects Customer Side swimlane flow.
  - Next: decompose Module 7 into `module/nomination-request/` (4-file structure) and proceed to `implementation/design/` for Customer Portal screens.

## 2026-04-25T21:12:07+07:00
- **Task Performed:** Created directory-level base-knowledge READMEs for `document/`, `module/`, and `implementation/` to guide future AI agents.
- **Files Modified:**
  - `document/README.md`
  - `module/README.md`
  - `implementation/README.md`
  - `work-log.md`
- **Logic / Decisions Made:**
  - Enforced the repo README’s directory boundaries and the `module/` 4-file invariant.
  - Added an explicit workflow pipeline for `/implementation/` (design → plan → replit-handoff) to reduce ambiguity for downstream agents.
- **Results / Next Steps:**
  - Agents can now navigate the repo at directory entry points.
  - Next: keep the `/document/` index updated as foundational files change.
