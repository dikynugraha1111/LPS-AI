# Replit Handoff — M13: Mother Vessel Master (Customer Portal)

**Version:** 1.0 (Mei 2026)

> ⚠ **STATUS CONTRACT STS: TENTATIVE.** Endpoint, payload, dan signature scheme STS Platform di handoff ini adalah **asumsi** berdasarkan briefing — belum dikonfirmasi tim STS (6 open questions, lihat akhir dokumen). Setiap titik asumsi ditandai **⚠ TENTATIVE**. Implementasi backend WAJIB pakai **adapter layer** (`internal/sts/asset_client.go`) sehingga contract bisa di-swap tanpa ubah handler/UI saat STS confirm. Frontend tidak terpengaruh (konsumsi endpoint LPS internal yang stabil).

## Context

Continuation of the LPS Customer Portal. **Module 13** memungkinkan customer mengelola data master **Mother Vessel (MV)** miliknya: list, Add, Edit, Deactivate. Master MV **otoritatif di STS Platform** — LPS **tidak menyimpan tabel master MV**, hanya proxy ke STS API + cache status approval (TTL singkat). Setiap aksi customer = **approval request** yang di-verifikasi STS Admin (lifecycle `PENDING_APPROVAL` → `APPROVED`/`REJECTED`). Hasil approval dikirim STS via **webhook** (push) ke LPS; polling/cache TTL sebagai fallback.

Phase 1: **MV only** — halaman khusus Mother Vessel, tanpa tabs/filter asset type lain.

### Data flow
1. Customer buka menu "Mother Vessel" → LPS proxy `GET /api/sts/assets?type=MV&owner_org_id=...` → tampil list.
2. Customer Add/Edit/Deactivate → LPS validasi → forward `POST /api/sts/assets/approval-request` → STS buat request status `PENDING` → LPS cache + return → row tampil badge "Pending Approval".
3. STS Admin approve/reject di STS Platform (di luar LPS).
4. STS push `POST /api/webhooks/sts/asset-approval-status` → LPS verifikasi HMAC → invalidate cache + in-app notification.
5. Customer lihat status terbaru (real-time via webhook; fallback cache TTL 600s / refresh manual).

## Prerequisites
- M7 complete: JWT auth, `customers` table, `organization_id` ada di JWT claim customer.
- System Config (M12): `STS_ASSET_API_KEY`, `STS_WEBHOOK_SECRET`, `STS_PLATFORM_BASE_URL`, `STS_APPROVAL_CACHE_TTL_SECONDS` (default 600).
- Reuse helper HMAC dari M9b: `verifySTSSignature` (jika sudah ada). Jika belum, implement di handoff ini (lihat Step 6).
- **Tidak ada dependency ke M8 untuk handoff ini.** Update M8 vessel dropdown = handoff terpisah.

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — **v1.4** (Modal Dialog, Confirmation Dialog, Form Field Grid, Audit Timeline, Disclaimer Banner, Locked Row Action — §3.2; status mapping `ACTIVE`/`INACTIVE`/`PENDING_APPROVAL`/`REJECTED` — §2.1).
2. **Per-modul UI design:** [`implementation/design/m13-mother-vessel-master-ui.md`](../design/m13-mother-vessel-master-ui.md) — sidebar placement, list page (search tanpa tabs, table 8 kolom, action icons kontekstual per status), modal Add/Edit (13 field di 3 grup), Deactivate confirmation, Detail + Riwayat Perubahan (Audit Timeline), edge cases, a11y notes.

**Surface:** A — Customer Portal (Bahasa Indonesia). Field label form pakai istilah maritim standar (Vessel Name, IMO, DWT, LOA — English, sesuai konvensi industri).

**UI rules ringkas:**
- shadcn/ui basis komponen. Styling Tailwind dari design system. Font Inter. Navy `#0F2A4D`. Canvas `bg-slate-50`.
- Sidebar A: nav item baru "Mother Vessel" (icon `Ship`) dalam grup label "MASTER DATA".
- Status badge: variant dari design system §2.1 v1.4 — `ACTIVE`→Confirmed, `INACTIVE`→Neutral, `PENDING_APPROVAL`→Pending verify, `REJECTED`→Error.
- Modal/Confirmation/Form Grid/Audit Timeline/Disclaimer Banner/Locked Row Action: pakai pattern §3.2 v1.4 **persis**.
- Inline validation **on blur** (bukan on keystroke, bukan submit-only). Required field asterisk rose. Disabled field `bg-slate-100 text-slate-500 cursor-not-allowed` + helper text.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate (hanya untuk cache table — TIDAK ada tabel master MV)
- **HTTP client:** Go `net/http` dengan timeout 3s + retry untuk call ke STS

---

## Step 1: Database Migration (cache only — NO master MV table)

File: `migrations/XXXXXX_create_asset_approval_cache_m13.up.sql`

> **PENTING:** LPS **TIDAK** membuat tabel `mother_vessels` / `assets`. Master MV milik STS. Migration ini HANYA tabel cache opsional + log idempotency webhook.

```sql
-- Cache status approval request (opsional — boleh skip jika STS API < 200ms; fallback: no-cache direct proxy)
CREATE TABLE IF NOT EXISTS asset_approval_cache (
    cache_key        VARCHAR(255) PRIMARY KEY,   -- "asset:{id}" | "asset:{id}:history" | "list:{org}:{hash}" | "active-mv:{org}"
    payload_json     JSONB NOT NULL,             -- cached response body
    cached_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at       TIMESTAMPTZ NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_asset_cache_expires ON asset_approval_cache(expires_at);

-- Idempotency log untuk webhook STS (cegah double-process)
CREATE TABLE IF NOT EXISTS sts_asset_webhook_log (
    request_id       VARCHAR(100) PRIMARY KEY,   -- dari payload webhook STS
    event            VARCHAR(60) NOT NULL,
    received_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> Jika tim memilih Redis untuk cache: tabel `asset_approval_cache` boleh diganti Redis key-value (TTL native). `sts_asset_webhook_log` tetap di Postgres (idempotency persisten).

---

## Step 2: STS Adapter Layer (⚠ TENTATIVE contract — isolasi di sini)

File: `internal/sts/asset_client.go`

**Tujuan:** Semua asumsi contract STS terisolasi di file ini. Saat STS confirm contract final, hanya file ini yang berubah — handler & frontend tidak.

```go
package sts

// ⚠ TENTATIVE — sesuaikan saat STS confirm (open questions #1–#6).
type AssetClient interface {
    ListAssets(ctx context.Context, ownerOrgID, status, search string, page, size int) (*AssetListResp, error)
    GetAsset(ctx context.Context, assetID string) (*Asset, error)
    GetApprovalHistory(ctx context.Context, assetID string) (*ApprovalHistoryResp, error)
    SubmitApprovalRequest(ctx context.Context, req SubmitApprovalReq) (*SubmitApprovalResp, error)
}

// Base URL + auth dari env: STS_PLATFORM_BASE_URL, STS_ASSET_API_KEY.
// Semua call: timeout 3s, retry 1x (GET) / 0x (POST — jangan double-submit), Bearer auth.
// ⚠ TENTATIVE endpoint paths:
//   GET  {BASE}/api/sts/assets?owner_org_id=&type=MV&status=&search=&page=&size=
//   GET  {BASE}/api/sts/assets/:id
//   GET  {BASE}/api/sts/assets/:id/approval-history
//   POST {BASE}/api/sts/assets/approval-request
```

Struct `Asset`, `AssetListResp`, `ApprovalHistoryResp`, `SubmitApprovalReq/Resp`: ikuti shape di [`module/mother-vessel-master/specifications.md`](../../module/mother-vessel-master/specifications.md) §3.1–§3.4. **Type=MV selalu di-hardcode** LPS-side (Phase 1 tidak pernah call type lain).

Provide mock impl `MockAssetClient` (in-memory) di belakang flag `STS_ASSET_MOCK=true` agar frontend bisa dikembangkan/tested sebelum STS contract final.

---

## Step 3: Backend — Customer Endpoints (proxy ke STS)

File: `internal/asset/handler.go`

Semua endpoint: Customer JWT. Extract `organization_id` **dari JWT** (bukan dari payload — security).

### 3.1 GET /api/customer/mother-vessels
Query: `status?`, `search?`, `page?`, `size?` (tidak ada `type` — hardcode `MV`).
1. Cek cache `list:{org}:{hash(query)}` (TTL 60–300s). Hit → return.
2. Miss → `assetClient.ListAssets(org, ...)`. Cache + return.
3. STS error 5xx/timeout → 503 `{error:"sts_unavailable"}`. Jika ada cache lama → return cache + header `X-Data-Stale: true` (frontend tampilkan caption "Data terakhir dimuat: ...").

### 3.2 GET /api/customer/mother-vessels/:id
Detail. Guard: response `owner_org_id == JWT.org` (reject 403 jika mismatch). Cache `asset:{id}` TTL 600s.

### 3.3 GET /api/customer/mother-vessels/:id/approval-history
Proxy `GetApprovalHistory`. Guard owner. Cache `asset:{id}:history` TTL 60s.

### 3.4 POST /api/customer/mother-vessels/approval-request
Body: `{ action: "ADD"|"EDIT"|"DEACTIVATE", asset_id: string|null, payload: {...} }`

Logic:
1. Validate JWT; extract `user_id`, `organization_id`.
2. Action constraints: `ADD` → `asset_id` null + payload required. `EDIT` → `asset_id` + full payload. `DEACTIVATE` → `asset_id` (payload boleh `{status:"INACTIVE"}`).
3. `EDIT`/`DEACTIVATE`: fetch asset dari STS → reject **403** jika `owner_org_id != JWT.org`; reject **409** jika current status `PENDING_APPROVAL` (lock — backup untuk UI lock).
4. Validate payload (struct tag / manual):
   - `vessel_name` (1–255), `imo_number` (regex `^\d{7}$`), `vessel_type` (non-empty), `call_sign` (1–50), `classification` (optional), `capacity_dwt`/`grt`/`length_loa_m`/`beam_m`/`max_draft_m` (numeric > 0), `status` (`ACTIVE`|`INACTIVE`).
   - ⚠ TENTATIVE: `vessel_type`/`classification` enum exhaustive list — open question #4. Untuk sekarang validasi non-empty saja; STS yang final-validate.
5. Inject `owner_org_id` dari JWT ke request STS (jangan percaya payload).
6. `assetClient.SubmitApprovalRequest(...)`.
7. Sukses → audit log (`MV_{ACTION}_REQUEST`, payload snapshot, sts `request_id`). Invalidate cache org. Return 201 STS response.
8. STS error: 400 → forward field errors. 422 → forward (mis. duplicate IMO). 409 → forward. 5xx/timeout → 503 `{error:"sts_unavailable", retry:true}`.

### 3.5 GET /api/customer/mother-vessels/active
**Untuk M8 (handoff terpisah).** Forward `ListAssets(org, status="ACTIVE", type=MV)`. Return ringkas `[{asset_id, asset_code, vessel_name, imo_number, capacity_dwt}]`. Cache `active-mv:{org}` TTL 60s. (Endpoint disiapkan di sini; konsumsi oleh M8 di handoff M8.)

---

## Step 4: Backend — STS Webhook Handler (⚠ TENTATIVE payload)

File: `internal/webhook/sts_asset_handler.go`

### POST /api/webhooks/sts/asset-approval-status

**Auth:** HMAC-SHA256 di `X-STS-Signature` (secret `STS_WEBHOOK_SECRET`). Reuse `verifySTSSignature` dari M9b.

```go
// ⚠ TENTATIVE payload — sesuaikan saat STS confirm (open question #3).
type STSAssetWebhookPayload struct {
    RequestID   string          `json:"request_id"`
    AssetID     string          `json:"asset_id"`
    OwnerOrgID  string          `json:"owner_org_id"`
    Event       string          `json:"event"` // ASSET_APPROVAL_APPROVED | ASSET_APPROVAL_REJECTED
    Timestamp   time.Time       `json:"timestamp"`
    Data        json.RawMessage `json:"data"`  // { action, asset_code, status, rejection_reason }
}
```

Processing:
1. Verifikasi HMAC. Invalid → **401**, jangan proses.
2. Idempotency: `INSERT ... ON CONFLICT (request_id) DO NOTHING` ke `sts_asset_webhook_log`. Jika sudah ada → return **200** no-op.
3. Invalidate cache: `asset:{asset_id}`, `asset:{asset_id}:history`, semua `list:{owner_org_id}:*`, `active-mv:{owner_org_id}`.
4. Create in-app notification (semua user di `owner_org_id`):
   - `ASSET_APPROVAL_APPROVED` + action ADD/EDIT → "MV {asset_code} Anda telah disetujui dan siap digunakan."
   - + action DEACTIVATE → "MV {asset_code} Anda telah dinonaktifkan."
   - `ASSET_APPROVAL_REJECTED` → "Pengajuan MV ditolak. Alasan: {rejection_reason}. Silakan perbaiki dan ajukan ulang."
5. Return **200** segera (notif boleh async).

> Webhook handler boleh di-deploy duluan walau STS belum kirim — idle sampai STS mulai. Sistem tetap jalan via fallback polling/cache TTL.

---

## Step 5: Frontend — Sidebar Nav

File: sidebar Customer Portal (mis. `src/components/layout/SidebarA.tsx`)

- Tambah grup label **"MASTER DATA"** (`text-xs font-semibold uppercase tracking-wider text-slate-400 px-3 mt-5 mb-1.5`) jika belum ada.
- Tambah nav item **"Mother Vessel"** icon `Ship`, link `/customer/mother-vessel`. Active state `bg-[#1E3A66] text-white`; inactive `text-slate-300 hover:bg-[#13315A] hover:text-white` (pattern Sidebar A design system §3.3).

---

## Step 6: Frontend — MV List Page

File: `src/pages/customer/mother-vessel/MotherVesselPage.tsx` · Route `/customer/mother-vessel`

Sesuai UI doc §4:
- PageHeader: title "Mother Vessel", subtitle "Kelola data Mother Vessel milik organisasi Anda. Setiap perubahan memerlukan persetujuan admin.", action primary `<Plus/> Add New MV`.
- Section Card: search bar (leading `Search` icon, rounded-lg, debounce 300ms, clear X) — **tanpa tabs/filter asset type**.
- Data Table (design system §3.2) 8 kolom: Aksi | Asset Code (mono) | Vessel Name | Tipe Kapal (title-case) | Classification | Capacity (ribuan titik, `tabular-nums`, right) | Owner | Status (badge §2.1 v1.4).
- Action icons per row (Icon-only `h-9 w-9`): View / Edit / Deactivate — perilaku kontekstual per status (UI doc §4.4):
  - View: selalu enabled.
  - Edit: enabled `ACTIVE`/`INACTIVE`/`REJECTED`; `PENDING_APPROVAL` → **Locked Row Action** (disabled opacity-40 + tooltip).
  - Deactivate: `ACTIVE` → Deactivate; `INACTIVE` → **Reactivate** (icon emerald, buka EditMVDialog prefilled Status=Active); `REJECTED` → disabled; `PENDING_APPROVAL` → Locked.
- Pagination: `Baris [10▾]` + counter + prev/next + refresh ⟳ (invalidate query).
- Empty/error/loading states per UI doc §4.6 (skeleton rows saat loading; banner error rose saat STS down + disable Add).
- Data: TanStack Query `useQuery(['mv-list', {search,page,size}])` → `GET /api/customer/mother-vessels`.

Status badge mapping helper:
```ts
const MV_STATUS = {
  ACTIVE:           { label: "Active",           variant: "confirmed" },
  INACTIVE:         { label: "Inactive",         variant: "neutral" },
  PENDING_APPROVAL: { label: "Pending Approval", variant: "pendingVerify" },
  REJECTED:         { label: "Rejected",         variant: "error" },
};
```

---

## Step 7: Frontend — Add / Edit MV Modal

File: `src/components/mother-vessel/MVFormDialog.tsx` (dipakai untuk Add & Edit)

Pakai **Modal Dialog** + **Form Field Grid** (design system §3.2 v1.4), `max-w-2xl`. Sesuai UI doc §5:
- Header: "Add New MV" / "Edit MV — {asset_code}" + subtitle + close X.
- **Disclaimer Banner** kuning di top body (severity warning §3.2 v1.4): "Semua perubahan dikirim sebagai permintaan persetujuan dan memerlukan review admin sebelum diterapkan."
- 13 field di grid 2-kolom, 3 grup subheading: **IDENTITAS** (Asset Code disabled, Vessel Name*, IMO*, Type*, Call Sign*, Classification), **DIMENSI & KAPASITAS** (Capacity DWT*, GRT*, Length LOA*, Beam*, Max Draft*, Status*), **KEPEMILIKAN** (Owner disabled).
- Asset Code & Owner: disabled (`bg-slate-100 text-slate-500 cursor-not-allowed` + helper text).
- Edit dari `REJECTED`: title "Edit & Ajukan Ulang — {asset_code}" + banner info biru di atas disclaimer: "MV ini sebelumnya ditolak. Alasan: {rejection_reason}. Perbaiki data lalu ajukan ulang."
- Validation **on blur**, error di bawah field (`text-rose-600 text-xs mt-1` + border rose). Submit disabled saat ada error/loading. Auto-focus field invalid pertama saat submit gagal.
- Submit → `POST /api/customer/mother-vessels/approval-request`. Feedback (UI doc §5.4): loading spinner → sukses toast + close + refetch list; 422 inline IMO error; 409 toast lock; 5xx toast + preserve input.
- Dismiss dengan unsaved changes → Confirmation Dialog "Tutup tanpa menyimpan?".

Footer: [Batal] (outlined) + [Submit for Approval] (primary navy).

---

## Step 8: Frontend — Deactivate Confirmation

File: `src/components/mother-vessel/DeactivateMVDialog.tsx`

Pakai **Confirmation Dialog** (design system §3.2 v1.4), `max-w-md`. Sesuai UI doc §6:
- Icon bulat amber `bg-amber-50 + AlertTriangle text-amber-600` (state change butuh approval — bukan delete destruktif).
- Copy: "Nonaktifkan MV {vessel_name}? Status akan diubah menjadi Inactive setelah disetujui admin STS. MV tidak akan muncul di pilihan nominasi."
- Tombol konfirmasi "Submit for Approval" (primary navy, **bukan** destructive rose). Default focus **Batal**.
- Submit → `POST .../approval-request` `{action:"DEACTIVATE", asset_id}`. Feedback sama pola Step 7.

---

## Step 9: Frontend — MV Detail + Riwayat Perubahan

File: `src/components/mother-vessel/MVDetailDialog.tsx` + route fallback `src/pages/customer/mother-vessel/MotherVesselDetailPage.tsx` (`/customer/mother-vessel/:id`)

Pakai **Modal Dialog** (detail variant `max-w-3xl`). Default modal; halaman penuh untuk deep-link/refresh. Sesuai UI doc §7:
- Header: "Detail MV — {vessel_name}" + sub "Asset Code {asset_code}" + Status Badge.
- Grup "DATA KAPAL": `<dl>` 2-kolom read-only semua field MV (Capacity/GRT ribuan titik `tabular-nums`; dimensi + unit).
- Grup "RIWAYAT PERUBAHAN": **Audit Timeline** (design system §3.2 v1.4) dari `GET /api/customer/mother-vessels/:id/approval-history`. Descending. Action label ID ("Pengajuan MV Baru"/"Perubahan Data"/"Nonaktifkan"), submitter+timestamp, status badge, reviewer+resolved timestamp, rejection reason di box rose. Empty state "Belum ada riwayat perubahan."
- Footer kontekstual: `REJECTED` → banner rose + [Edit & Ajukan Ulang] (buka MVFormDialog prefilled) + [Tutup]; `PENDING_APPROVAL` → [Tutup] + caption "Pengajuan sedang ditinjau admin."; lainnya → [Tutup].

---

## Step 10: Frontend — Notifications (webhook result)

In-app notification dari webhook (Step 4). Reuse infra notifikasi LPS existing (mis. notification bell / toast on login). Saat customer buka portal & ada notif baru terkait MV → tampilkan. Tidak perlu polling khusus M13 — webhook + cache invalidation sudah handle; list akan fresh saat di-refetch (cache sudah di-invalidate webhook).

---

## Acceptance Checklist

### Backend — Data Ownership & Proxy
- [ ] **TIDAK ada** tabel master MV di LPS DB. Hanya `asset_approval_cache` (opsional) + `sts_asset_webhook_log`.
- [ ] STS contract terisolasi di `internal/sts/asset_client.go` (adapter). `MockAssetClient` tersedia di belakang `STS_ASSET_MOCK=true`.
- [ ] `GET /api/customer/mother-vessels` proxy ke STS, `type=MV` hardcoded, `owner_org_id` dari JWT (bukan payload).
- [ ] `GET /:id` + `/:id/approval-history` guard owner (403 jika org mismatch).
- [ ] `POST /approval-request`: action constraints, payload validation (IMO 7 digit, numeric > 0), 409 jika current `PENDING_APPROVAL`, audit log, cache invalidation.
- [ ] `GET /active` return MV `ACTIVE` ringkas (untuk M8 nanti).
- [ ] STS down → 503 + (jika ada) cache stale + `X-Data-Stale: true`. Timeout 3s, retry 1x GET / 0x POST.

### Backend — Webhook
- [ ] `POST /api/webhooks/sts/asset-approval-status` verifikasi HMAC (`STS_WEBHOOK_SECRET`); invalid → 401.
- [ ] Idempotent via `sts_asset_webhook_log` (ON CONFLICT DO NOTHING) → 200 no-op jika duplikat.
- [ ] Invalidate cache (`asset:{id}`, `:history`, `list:{org}:*`, `active-mv:{org}`) + create in-app notification.
- [ ] Return 200 cepat; handler tetap berfungsi walau STS belum kirim (idle).

### Frontend — List
- [ ] Sidebar: grup "MASTER DATA" + item "Mother Vessel" (icon Ship), active state benar.
- [ ] List page: search debounce 300ms, **tanpa tabs/filter asset type**, table 8 kolom, badge §2.1 v1.4.
- [ ] Action icons kontekstual: View selalu aktif; Edit locked saat `PENDING_APPROVAL`; Deactivate→Reactivate saat `INACTIVE`, disabled saat `REJECTED`, locked saat `PENDING_APPROVAL`.
- [ ] Pagination + refresh; empty (belum ada / search no-match) + error (STS down, disable Add) + loading (skeleton) states.

### Frontend — Modals
- [ ] Add/Edit: Modal Dialog + Form Field Grid, 13 field 3 grup, Asset Code & Owner disabled, Disclaimer Banner kuning.
- [ ] Edit dari `REJECTED`: title "Edit & Ajukan Ulang" + banner info biru alasan reject.
- [ ] Validation on blur, error di bawah field, auto-focus invalid pertama, submit disabled saat error/loading.
- [ ] Submit feedback: loading→toast sukses+close+refetch; 422 inline IMO; 409 toast lock; 5xx toast+preserve.
- [ ] Dismiss unsaved → confirmation "Tutup tanpa menyimpan?".
- [ ] Deactivate: Confirmation Dialog amber, copy konsekuensi konkret, tombol "Submit for Approval" navy (bukan rose), default focus Batal.
- [ ] Detail: Modal `max-w-3xl`, data kapal read-only, Audit Timeline descending dengan rejection reason box rose, footer kontekstual per status.

### A11y
- [ ] Icon-only button `aria-label` deskriptif; locked button aria-label nyatakan status terkunci.
- [ ] Modal focus trap, Esc close, focus ke field/heading saat open & balik ke trigger saat close.
- [ ] Form `<label htmlFor>` semua field (no placeholder-only); error `role="alert"`/aria-live.
- [ ] Status badge warna + teks (bukan warna saja). Kontras AA. Toast `aria-live="polite"` tidak rebut focus.

---

## ⚠ Open Questions — Tim STS (blokir finalisasi contract)

Handoff ini pakai asumsi. Sebelum production, konfirmasi tim STS lalu update `internal/sts/asset_client.go` + `module/mother-vessel-master/specifications.md` §3:

1. **Endpoint paths final** STS (list/detail/history/submit) — saat ini asumsi `/api/sts/assets*`.
2. **Asset Code generation** — STS generate saat APPROVED? Format `MV-XXXX`? Ada draft code saat PENDING?
3. **Webhook payload & signature** — field exact, scheme HMAC (header name, body canonicalization) — saat ini asumsi pola M9b.
4. **Validasi field STS-side** — enum exhaustive `vessel_type` & `classification`, aturan IMO uniqueness.
5. **Rate limit** STS untuk approval request submissions per org.
6. **Approval history shape** — `payload_diff` (diff) atau full snapshot per entry (pengaruh tampilan Riwayat Perubahan).

> Sampai resolved: jalankan dengan `STS_ASSET_MOCK=true` untuk dev/test frontend. Backend proxy + webhook handler boleh deploy (idle) — sistem tidak crash, fallback cache/polling aktif.
