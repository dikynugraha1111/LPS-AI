# M13 — Mother Vessel Master (Customer Portal): Technical Specifications

> **v1.0 (Mei 2026):** Modul BARU. Data master MV otoritatif di STS Platform. LPS sebagai UI proxy + cache status approval request (TTL singkat). Phase 1: MV only.

> **v1.1 (Mei 2026):** Notifikasi hasil approval pakai **webhook STS → LPS (push)** sebagai mekanisme utama (pola HMAC seperti M9b), dengan polling/cache TTL sebagai fallback. Tambah §3.5 webhook inbound + §4.4 endpoint webhook. Lifecycle §1 diupdate.

## 1. Approval Request Lifecycle

```
Customer aksi di Customer Portal (Add / Edit / Deactivate)
      │
      ▼
LPS endpoint  POST /api/customer/mother-vessels/approval-request
      │
      │ 1. Validate JWT customer
      │ 2. Validate owner_org_id == JWT.organization_id
      │ 3. Validate payload (Zod / schema)
      │ 4. Forward ke STS API
      ▼
STS API  POST /api/sts/assets/approval-request
      │
      │ → Create approval request — status PENDING
      │ → Return { request_id, asset_id_temp_or_existing, status: PENDING }
      ▼
LPS cache approval status (key = asset_id atau request_id, TTL 600s)
Return 201 ke customer dengan status PENDING_APPROVAL
      │
      ▼
[Out of LPS scope]
STS Admin verifikasi di STS Platform admin UI:
      ├─ Approve → STS update master MV
      └─ Reject  → STS attach rejection_reason
      │
      ▼
STS push webhook → LPS  POST /api/webhooks/sts/asset-approval-status
      │ (HMAC-SHA256 verified)
      │ → LPS invalidate cache untuk asset_id/org terkait
      │ → LPS create in-app notification ke customer
      ▼
Customer melihat status terbaru (real-time via webhook; fallback: cache expire 600s / manual refresh)
```

**Mekanisme notifikasi (v1.1):**
- **Utama — Webhook push:** STS kirim `POST /api/webhooks/sts/asset-approval-status` saat approval di-resolve. LPS verifikasi HMAC, invalidate cache, buat in-app notification. Lihat §3.5.
- **Fallback — Polling/cache TTL:** jika webhook gagal/terlambat, status tetap ter-refresh saat cache TTL habis (600s) atau customer klik refresh ⟳. Menjamin eventual consistency walau webhook drop.

**Status enum (dari STS, di-display di LPS):**
- `ACTIVE` — MV aktif, dapat dipakai di nominasi
- `INACTIVE` — MV deactivated (approved)
- `PENDING_APPROVAL` — submission menunggu STS Admin (akan dijadikan UI label "Pending Approval")
- `REJECTED` — STS Admin menolak; `rejection_reason` tersedia

**UI label mapping (Bahasa Indonesia + English label sesuai screenshot):**
- `ACTIVE` → "Active" (badge hijau)
- `INACTIVE` → "Inactive" (badge abu-abu)
- `PENDING_APPROVAL` → "Pending Approval" (badge kuning)
- `REJECTED` → "Rejected" (badge merah)

## 2. Data Ownership

**LPS DB:** **TIDAK** menyimpan tabel `mother_vessels`. Tidak ada migration untuk master MV. Yang LPS simpan hanya:

```sql
-- Cache approval request status (key-value cache lokal)
-- Implementasi bisa Redis, atau tabel SQL ringan kalau Redis belum ada
-- TTL: 600 detik (10 menit). Invalidate manual saat customer baru submit aksi.
CREATE TABLE asset_approval_cache (
    cache_key       VARCHAR(255) PRIMARY KEY,  -- format: "asset:{asset_id}" atau "request:{request_id}"
    status          VARCHAR(40) NOT NULL,      -- PENDING_APPROVAL | APPROVED | REJECTED
    rejection_reason TEXT,
    cached_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_asset_approval_cache_expires ON asset_approval_cache(expires_at);
```

> Cache opsional — Replit Agent boleh skip kalau STS API cukup cepat (< 200ms). Implementasi paling sederhana: tidak ada cache, semua GET langsung proxy ke STS dengan timeout 3s.

**Audit log LPS (sudah ada dari M12):**
- Setiap submission approval request dari LPS ter-log di tabel `audit_logs` (atau equivalent):
  - `actor_id` (customer user)
  - `action` (`MV_ADD_REQUEST`, `MV_EDIT_REQUEST`, `MV_DEACTIVATE_REQUEST`)
  - `payload_json` (snapshot data submission)
  - `sts_request_id` (return dari STS)
  - `timestamp`

## 3. STS Platform API Contract (Outbound)

Detail final endpoint **menunggu konfirmasi tim STS**. Berikut asumsi contract berdasarkan briefing:

### 3.1 List Assets (GET)

```
GET {STS_BASE_URL}/api/sts/assets?owner_org_id={oid}&type={type}&status={status}&page={n}&size={s}
Headers:
  Authorization: Bearer {STS_API_KEY}
  X-LPS-Customer-Id: {customer_id from JWT}

Query params:
  owner_org_id  (required) — auto-set dari JWT customer.organization_id
  type          (required) — selalu `MV` di Phase 1 (LPS hardcode; STS API mungkin support type lain untuk konsumen lain)
  status        (optional) — ACTIVE | INACTIVE | PENDING_APPROVAL | REJECTED
  page, size    (optional) — pagination
```

> **Phase 1:** LPS selalu mengirim `type=MV`. Endpoint tidak pernah memanggil STS untuk asset type lain karena halaman khusus Mother Vessel.

Response:
```json
{
  "items": [
    {
      "asset_id": "uuid",
      "asset_code": "MV-0001",
      "type": "MV",
      "vessel_name": "MV Karimun Jawa",
      "vessel_type": "Gear",
      "imo_number": "9876543",
      "call_sign": "PXTK",
      "classification": "Panamax",
      "capacity_dwt": 50000,
      "grt": 30000,
      "length_loa_m": 189,
      "beam_m": 32,
      "max_draft_m": 12.5,
      "owner_org_id": "uuid",
      "owner_name": "PT Adaro Energy Indonesia",
      "status": "ACTIVE",
      "rejection_reason": null,
      "last_request_id": "uuid",
      "created_at": "2026-01-15T08:00:00Z",
      "updated_at": "2026-04-20T10:30:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "size": 10
}
```

### 3.2 Get Asset Detail (GET)

```
GET {STS_BASE_URL}/api/sts/assets/:asset_id
Headers: Authorization: Bearer {STS_API_KEY}
```

Response: single asset object (same shape as items array).

### 3.3 Submit Approval Request (POST)

```
POST {STS_BASE_URL}/api/sts/assets/approval-request
Headers:
  Authorization: Bearer {STS_API_KEY}
  Content-Type: application/json
  X-LPS-Customer-Id: {customer_id from JWT}
  X-LPS-User-Id: {user_id from JWT}

Body:
{
  "action": "ADD" | "EDIT" | "DEACTIVATE",
  "asset_id": "uuid" | null,        // null untuk ADD; required untuk EDIT/DEACTIVATE
  "owner_org_id": "uuid",            // dari JWT, untuk validasi double-check di STS
  "payload": {
    // Untuk ADD/EDIT — full field set MV:
    "type": "MV",
    "vessel_name": "MV Pacific Star",
    "imo_number": "1234567",
    "vessel_type": "Bulk Carrier",
    "call_sign": "PXTS",
    "classification": "BKI",
    "capacity_dwt": 75000,
    "grt": 45000,
    "length_loa_m": 220,
    "beam_m": 35,
    "max_draft_m": 14.0,
    "status": "ACTIVE"
    // Untuk DEACTIVATE — minimal:
    // (asset_id sudah di top-level; payload bisa kosong atau { "status": "INACTIVE" })
  },
  "submitted_by_user_id": "uuid",
  "submitted_at": "ISO-8601"
}
```

Response:
```json
{
  "request_id": "uuid",
  "asset_id": "uuid",  // existing untuk EDIT/DEACTIVATE; temp/draft untuk ADD
  "status": "PENDING_APPROVAL",
  "submitted_at": "ISO-8601"
}
```

Errors:
- `400` — Payload validation gagal (field missing, format salah).
- `403` — `owner_org_id` mismatch dengan JWT (LPS reject sebelum forward, atau STS reject).
- `409` — Sudah ada pending request untuk `asset_id` ini (lock check di STS — backup untuk LPS-side lock).
- `422` — Business rule reject (e.g. duplicate IMO).
- `502` — STS unreachable.

### 3.4 Approval History (GET)

```
GET {STS_BASE_URL}/api/sts/assets/:asset_id/approval-history
Headers: Authorization: Bearer {STS_API_KEY}
```

Response:
```json
{
  "asset_id": "uuid",
  "history": [
    {
      "request_id": "uuid",
      "action": "EDIT",
      "submitted_at": "2026-05-10T08:00:00Z",
      "submitter": { "user_id": "uuid", "name": "Ishac Dainurry" },
      "status": "REJECTED",
      "reviewer": { "user_id": "uuid", "name": "STS Admin A" },
      "rejection_reason": "IMO Number tidak valid",
      "resolved_at": "2026-05-10T14:32:00Z",
      "payload_diff": { /* JSON diff dari sebelumnya, opsional */ }
    },
    {
      "request_id": "uuid",
      "action": "ADD",
      "submitted_at": "2026-04-01T09:00:00Z",
      "submitter": { "user_id": "uuid", "name": "Ishac Dainurry" },
      "status": "APPROVED",
      "reviewer": { "user_id": "uuid", "name": "STS Admin B" },
      "rejection_reason": null,
      "resolved_at": "2026-04-01T11:00:00Z"
    }
  ]
}
```

### 3.5 STS Webhook — Inbound (v1.1)

> ⚠ **TENTATIVE** — payload & signature scheme menunggu konfirmasi tim STS (open question §11). Pola mengikuti webhook M9b (HMAC-SHA256).

#### Endpoint: POST /api/webhooks/sts/asset-approval-status

**Auth:** HMAC-SHA256 signature di header `X-STS-Signature` (shared secret `STS_WEBHOOK_SECRET`). Verifikasi reuse helper dari M9b (`verifySTSSignature`).

**Payload (asumsi):**
```json
{
  "request_id": "uuid",
  "asset_id": "uuid",
  "owner_org_id": "uuid",
  "event": "ASSET_APPROVAL_APPROVED | ASSET_APPROVAL_REJECTED",
  "timestamp": "ISO-8601",
  "data": {
    "action": "ADD | EDIT | DEACTIVATE",
    "asset_code": "MV-0042",          // final code (saat APPROVED untuk ADD)
    "status": "ACTIVE | INACTIVE",     // status master MV setelah resolve
    "rejection_reason": "..."          // hanya saat REJECTED
  }
}
```

**Processing:**
1. Verifikasi HMAC. Invalid → 401 (jangan proses).
2. Idempotency: jika `request_id` sudah diproses (cek cache/log), return 200 no-op.
3. Invalidate cache: `asset:{asset_id}`, `asset:{asset_id}:history`, list cache `(owner_org_id, ...)`, `active-mv:{owner_org_id}`.
4. Create in-app notification ke customer (semua user di `owner_org_id`):
   - APPROVED: "MV {asset_code} Anda telah disetujui dan siap digunakan." (atau "dinonaktifkan" untuk DEACTIVATE)
   - REJECTED: "Pengajuan MV ditolak. Alasan: {rejection_reason}. Silakan perbaiki dan ajukan ulang."
5. Return 200 OK segera (proses notif async bila perlu).

> Jika STS belum siap kirim webhook saat go-live: sistem tetap berfungsi via fallback polling/cache TTL (§1). Webhook handler bisa di-deploy duluan (idle sampai STS mulai kirim).

## 4. LPS API Endpoints (Customer-facing)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | `/api/customer/mother-vessels` | List assets milik customer (proxy ke STS, filter `owner_org_id` auto dari JWT) | Customer JWT |
| GET | `/api/customer/mother-vessels/:id` | Detail asset (proxy ke STS) | Customer JWT |
| GET | `/api/customer/mother-vessels/:id/approval-history` | Riwayat approval request (proxy ke STS) | Customer JWT |
| POST | `/api/customer/mother-vessels/approval-request` | Submit Add/Edit/Deactivate approval request (forward ke STS) | Customer JWT |
| GET | `/api/customer/mother-vessels/active` | List MV ber-status `ACTIVE` milik customer — **dipakai oleh M8 vessel dropdown** | Customer JWT |
| POST | `/api/webhooks/sts/asset-approval-status` | Inbound webhook STS — hasil approval (APPROVED/REJECTED). Invalidate cache + notif. Lihat §3.5 | HMAC |

### 4.1 GET /api/customer/mother-vessels

Query params:
- `status` (optional): filter by status enum (`ACTIVE` | `INACTIVE` | `PENDING_APPROVAL` | `REJECTED`)
- `search` (optional): keyword match terhadap `asset_code`, `vessel_name`, `imo_number`, `owner_name`
- `page`, `size` (pagination)

> Tidak ada query param `type` — Phase 1 endpoint ini **selalu MV**. LPS hardcode `type=MV` saat call STS.

Logic:
1. Extract `organization_id` dari JWT customer.
2. Forward ke STS `GET /api/sts/assets?owner_org_id={organization_id}&type=MV&...` dengan timeout 3s + retry 1×.
3. Cache response per `(organization_id, status, search, page, size)` di local cache TTL 60–300 detik (lebih singkat dari approval cache).
4. Return raw STS response.

### 4.2 POST /api/customer/mother-vessels/approval-request

Body (dari frontend):
```json
{
  "action": "ADD" | "EDIT" | "DEACTIVATE",
  "asset_id": "uuid" | null,
  "payload": { /* MV field set */ }
}
```

Logic:
1. Validate JWT customer; extract `user_id`, `organization_id`.
2. **Validate action constraints:**
   - `ADD`: `asset_id` harus `null`. Payload required.
   - `EDIT`: `asset_id` required. Payload required (full field set).
   - `DEACTIVATE`: `asset_id` required. Payload bisa kosong atau `{ status: "INACTIVE" }`.
3. **For EDIT/DEACTIVATE:** Fetch asset detail dari STS. Reject 403 jika `owner_org_id != JWT.organization_id`. Reject 409 jika current status sudah `PENDING_APPROVAL`.
4. **For ADD:** Pastikan `payload.imo_number` belum exist di STS (STS validasi final, LPS validasi early).
5. Validate payload field-level (Zod schema):
   - `vessel_name` (string, 1-255)
   - `imo_number` (string, exactly 7 digits)
   - `vessel_type` (enum, dari list yang disepakati STS)
   - `call_sign` (string, 1-50)
   - `classification` (string, optional)
   - `capacity_dwt`, `grt`, `length_loa_m`, `beam_m`, `max_draft_m` (numeric, > 0)
   - `status` (`ACTIVE` | `INACTIVE`)
6. Forward ke STS `POST /api/sts/assets/approval-request` dengan auth Bearer.
7. On STS success: log audit (action, payload, request_id, timestamp). Invalidate list cache untuk `organization_id` ini.
8. On STS error (400/422): forward error message ke customer. On 502: return 503 dengan retry hint.
9. Return STS response shape ke client.

### 4.3 GET /api/customer/mother-vessels/active

**Khusus untuk dipakai oleh M8 Nomination form (vessel dropdown).**

Logic:
1. Forward ke STS `GET /api/sts/assets?owner_org_id={organization_id}&type=MV&status=ACTIVE`.
2. Return ringkas: `[{ asset_id, asset_code, vessel_name, imo_number, capacity_dwt }]`.

> Endpoint terpisah dari general listing untuk:
> - **Performance:** payload lebih ringan.
> - **Filter eksplisit:** memastikan dropdown M8 tidak pernah menampilkan MV `PENDING_APPROVAL`, `INACTIVE`, atau `REJECTED`.

## 5. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/customer/mother-vessel` | `MotherVesselPage` | List view MV (search, table, pagination, "Add New MV" button). **Tanpa tabs filter asset type.** |
| `/customer/mother-vessel/:id` | `MotherVesselDetailPage` atau modal `MotherVesselDetailDialog` | Detail MV + section Riwayat Perubahan |

**Modal/Dialog (tidak punya route khusus):**
- `AddMVDialog` — dipicu klik "Add New MV"
- `EditMVDialog` — dipicu klik icon pencil di row
- `DeactivateMVDialog` — dipicu klik icon power di row (konfirmasi sederhana)

## 6. Sidebar Menu (Customer Portal)

Menu sidebar tambahan di section "MASTER DATA" (atau group baru sesuai design):

```
MASTER DATA
  🚢 Mother Vessel        ← link ke /customer/mother-vessel
```

> Mengikuti pattern Surface A sidebar yang sudah ada di Customer Portal. Posisi exact di sidebar ditentukan oleh per-modul UI design doc.

## 7. M8 Integration — Vessel Dropdown Filter

**File yang perlu diubah di M8:** [`module/nomination-submission/specifications.md`](../nomination-submission/specifications.md) dan replit handoff terkait.

**Perubahan:**
- Form Nomination M8 saat ini (asumsi) fetch list vessel dari endpoint lama (mungkin `GET /api/customer/vessels` atau hardcoded). 
- **Diganti** ke `GET /api/customer/mother-vessels/active` agar dropdown hanya menampilkan MV ber-status `ACTIVE` milik customer.
- Validasi server-side di M8 submit endpoint: pastikan `vessel_id` yang dikirim valid dan `ACTIVE` (re-check ke STS via cached call atau direct).

## 8. Caching Strategy

| Cache Key | TTL | Invalidation Trigger |
|-----------|-----|----------------------|
| List `(org_id, status, search, page, size)` | 60–300 s | Customer submit approval request di org tsb · **webhook STS resolve (§3.5)** |
| Detail `asset:{asset_id}` | 600 s | Submit approval request untuk asset tsb · **webhook STS resolve** |
| Approval history `asset:{asset_id}:history` | 60 s | Submit approval request untuk asset tsb · **webhook STS resolve** |
| Active MV list (M8 consumer) `(org_id):active-mv` | 60 s | Submit approval request · **webhook STS resolve** · periodic refresh |

> Webhook STS (§3.5) meng-invalidate cache secara proaktif → status terbaru tampil hampir real-time. TTL pendek tetap dipertahankan sebagai fallback bila webhook drop.

Implementasi cache: Redis recommended; fallback in-memory LRU per Echo handler. Tidak boleh long TTL — karena status approval bisa berubah cepat saat STS Admin online.

## 9. Error Handling

| Scenario | Response | UI Behavior |
|----------|----------|-------------|
| STS API 5xx | 503 dari LPS | Banner "Layanan master vessel sedang tidak tersedia. Coba lagi nanti." Disable tombol Add/Edit/Deactivate. List view tampil dari cache (jika ada). |
| STS API timeout (>3s) | 504 dari LPS | Sama dengan 5xx. Auto-retry 1× di backend; jika tetap fail return error ke client. |
| Validation error 400 | 400 dengan detail field | Form inline errors per field (Zod errors). |
| Owner mismatch 403 | 403 | Toast "Anda tidak memiliki akses untuk mengelola asset ini." (defensive — UI seharusnya tidak menampilkan tombol jika tidak owner). |
| Pending lock conflict 409 | 409 | Toast "Asset ini sedang menunggu approval admin. Edit/Deactivate akan tersedia setelah approval selesai." Disable tombol di UI. |
| Duplicate IMO 422 | 422 | Inline error di field IMO: "Nomor IMO sudah terdaftar." |
| Network error (LPS unreachable) | — | Toast generic + retry button. Form data preserved di state. |

## 10. Security & Privacy

- **JWT validation:** Semua endpoint customer-facing wajib customer JWT. Extract `organization_id` selalu dari JWT, **tidak** dari payload (mencegah customer manipulate `owner_org_id`).
- **STS API key:** `STS_API_KEY` di env, **tidak boleh** di-expose ke browser. Semua call ke STS dari server-side (Echo handler).
- **Payload size limit:** Body max 50 KB (form MV kecil, defensive limit).
- **Rate limiting:** Suggest 30 approval request submissions per customer per jam (configurable). Mencegah spam ke STS approval queue.

## 11. Open Questions (Untuk Konfirmasi Tim STS)

1. Final endpoint paths STS API (asumsi di atas tentative).
2. Format `asset_code` (apakah STS yang generate setelah approve, atau ada draft code?).
3. Apakah STS push webhook ke LPS saat approval result tersedia (push-based) atau LPS perlu polling?
4. Validasi field STS-side (IMO uniqueness rule, vessel_type enum exhaustive list, classification enum).
5. Rate limit STS side untuk approval request submissions.
6. Apakah `EDIT` request menyimpan diff atau full snapshot? (Mempengaruhi UI Riwayat Perubahan.)

---

> **Update file ini bersamaan dengan setiap perubahan API contract STS.** Setelah confirmasi tim STS, update section 3 dengan endpoint final + remove "Asumsi" disclaimer.
