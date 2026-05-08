# Replit Handoff — M10: Customer Dashboard & Monitoring

## Context
Continuation of the LPS Customer Portal. **Module 10** provides the customer landing dashboard, full nomination list, voyage tracking with AIS map, and weather/alert viewing. This module is purely read-only — it aggregates data from M8, M9, the LPS AIS system, and the LPS Weather cache. Requires M7, M8, M9 to be complete.

## Prerequisites
- M7, M8, M9 complete.
- LPS Weather cache API available at internal endpoint (see Step 3).
- LPS AIS data available at internal endpoint (see Step 4).
- Env var: `LPS_INTERNAL_API_KEY` for accessing internal LPS services.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, Recharts (charts), Framer Motion (animations), wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL (read-only queries)
- **Map:** Use Leaflet.js via `react-leaflet` for vessel position map

---

## Step 1: Backend — Dashboard Aggregation Endpoint

File: `internal/customer/dashboard_handler.go`

### GET /api/customer/dashboard
Auth: Customer JWT

Returns a single aggregated response to minimize frontend round-trips:

```json
{
  "active_nominations": [
    {
      "id": "uuid",
      "nomination_number": "NOM-20260425-0001",
      "vessel_name": "MV Example",
      "eta": "ISO-8601",
      "status": "PENDING",
      "status_label": "Menunggu proses di STS Platform",
      "updated_at": "ISO-8601"
    }
  ],
  "active_voyages": [
    {
      "nomination_id": "uuid",
      "nomination_number": "NOM-20260425-0001",
      "vessel_name": "MV Example",
      "anchor_point": "AP-3",
      "etb": "ISO-8601",
      "ais_last_position": {
        "lat": -3.8123,
        "lng": 115.9876,
        "last_updated": "ISO-8601"
      }
    }
  ],
  "weather": {
    "wave_height_m": 1.8,
    "wind_speed_kmh": 22,
    "condition": "NORMAL",
    "updated_at": "ISO-8601"
  }
}
```

**Query logic:**
- `active_nominations`: nominations WHERE `customer_id = :customer_id` AND `status NOT IN ('DRAFT', 'PAYMENT_CONFIRMED', 'SUBMIT_FAILED')` ORDER BY `updated_at DESC` LIMIT 10.
- `active_voyages`: nominations WHERE `customer_id = :customer_id` AND `status = 'PAYMENT_CONFIRMED'`. For each, fetch latest AIS position from internal AIS API.
- `weather`: fetch from LPS weather cache (see Step 3).

**Status label mapping (Go):**
```go
// Status pada tabel nominations (BRD v3.1):
// APPROVED = nominasi disetujui, EPB tersedia, belum ada bukti pembayaran.
// WAITING_PAYMENT_VERIFICATION = customer sudah upload proof di M9b, menunggu keputusan STS.
// Detail payment (proof, rejection reason) ada di tabel epb_payments (M9b).
var statusLabels = map[string]string{
    "DRAFT":                         "Draft",
    "SUBMITTED":                     "Submitted",
    "PENDING":                       "Menunggu proses di STS Platform",
    "APPROVED":                      "Nominasi Disetujui",
    "NEED_REVISION":                 "Perlu Revisi",
    "WAITING_PAYMENT_VERIFICATION":  "Menunggu Verifikasi Pembayaran",
    "PAYMENT_CONFIRMED":             "Pembayaran Dikonfirmasi",
    "SUBMIT_FAILED":                 "Gagal Dikirim",
}
```

---

## Step 2: Backend — Nominations List Endpoint

### GET /api/customer/nominations
Auth: Customer JWT. Returns all nominations for authenticated customer.

Query params:
- `?filter=active` → status NOT IN ('DRAFT', 'PAYMENT_CONFIRMED', 'SUBMIT_FAILED')
- `?filter=draft` → status = 'DRAFT'
- `?filter=completed` → status = 'PAYMENT_CONFIRMED'
- (no filter) → all

> Nominasi dengan status `APPROVED` yang sudah submit EPB Confirmation tetap tampil di filter `active`. Detail payment tersedia di menu EPB & Invoice (M9b).

Response: array of nomination summaries (id, nomination_number, vessel_name, eta, status, status_label, created_at).
Order: `created_at DESC`.

---

## Step 3: Backend — Weather Endpoint for Customer

### GET /api/customer/weather
Auth: Customer JWT

Fetch from LPS weather cache (internal API — same data shown to operators):
```go
// Call internal LPS weather service
GET http://localhost:{INTERNAL_PORT}/internal/weather/current
Headers: X-Internal-Key: {LPS_INTERNAL_API_KEY}
```

If internal API unavailable, return last cached value from `weather_cache` table (read-only).

Response:
```json
{
  "current": {
    "wave_height_m": 1.8,
    "wind_speed_kmh": 22,
    "visibility_km": 10,
    "condition": "NORMAL",
    "updated_at": "ISO-8601"
  },
  "alerts": [
    {
      "id": "uuid",
      "level": "WARNING",
      "message": "Tinggi gelombang mendekati batas aman (2.3m)",
      "created_at": "ISO-8601",
      "resolved_at": null
    }
  ]
}
```

`alerts` = last 24h from `alerts` table WHERE `category = 'WEATHER'`, ordered by `created_at DESC`.

`condition` logic:
- `wave_height_m < 2.5` → `NORMAL`
- `wave_height_m >= 2.5 AND < 3.0` → `WARNING`
- `wave_height_m >= 3.0` → `CRITICAL`

---

## Step 4: Backend — Voyage Tracking Endpoint

### GET /api/customer/voyages/:nomination_id
Auth: Customer JWT

1. Find nomination by ID. Guard: must belong to customer AND status = `PAYMENT_CONFIRMED`.
2. Fetch AIS position for vessel:
```go
GET http://localhost:{INTERNAL_PORT}/internal/ais/vessel?name={vessel_name}
Headers: X-Internal-Key: {LPS_INTERNAL_API_KEY}
```
3. Return combined response:
```json
{
  "nomination_id": "uuid",
  "nomination_number": "",
  "vessel_name": "",
  "anchor_point": "AP-3",
  "etb": "ISO-8601",
  "estimated_duration_hours": 36,
  "ais_position": {
    "lat": -3.8123,
    "lng": 115.9876,
    "speed_knots": 4.2,
    "heading": 145,
    "last_updated": "ISO-8601"
  },
  "ais_stale": false
}
```

`ais_stale = true` if `last_updated` is older than 10 minutes.

If AIS data unavailable: return `ais_position: null` and `ais_stale: true`.

---

## Step 5: Frontend — Customer Dashboard Page

File: `src/pages/customer/DashboardPage.tsx`
Route: `/customer/dashboard`

Fetch: `GET /api/customer/dashboard` using TanStack React Query with `refetchInterval: 60000` (60s).

**Layout (3-column grid on desktop, stacked on mobile):**

**Column 1 — Active Nominations (full width on top):**
- Table: Nomination Number, Vessel Name, ETA, Status badge (color-coded per status), Last Updated
- "Buat Nominasi Baru" button → navigates to `/customer/nominations/new`
- Empty state: "Tidak ada nominasi aktif. Buat nominasi baru untuk memulai."

**Column 2 — Active Voyages:**
- Cards per voyage: Vessel Name, Anchor Point, ETB
- "Lacak Posisi" button → `/customer/voyages/:nomination_id`
- Empty state: "Tidak ada voyage aktif."

**Column 3 — Weather Widget:**
- Wave height (large number)
- Wind speed, Visibility
- Condition badge: green (NORMAL) / yellow (WARNING) / red (CRITICAL)
- "Lihat Detail Cuaca" link → `/customer/weather`

Use Framer Motion `fadeIn` animation on page load for each section card.

---

## Step 6: Frontend — Nomination List Page

File: `src/pages/customer/NominationListPage.tsx`
Route: `/customer/nominations`

Fetch: `GET /api/customer/nominations` (no filter = all).

**Header:** "Daftar Nominasi" + "Buat Nominasi Baru" button

**Filter tabs:** All | Active | Draft | Completed
Each tab passes `?filter=` query param and re-fetches.

**Table columns:** Nomination Number, Vessel Name, ETA, Status badge, Created At, Action
Action column: "Lihat Detail" → `/customer/nominations/:id`

**Status badge colors:**
- DRAFT → gray
- SUBMITTED / PENDING → blue
- APPROVED → green
- NEED_REVISION → orange
- WAITING_PAYMENT_VERIFICATION → yellow (label: "Menunggu Verifikasi Pembayaran")
- PAYMENT_CONFIRMED → dark green
- SUBMIT_FAILED → red

Empty state per filter: "Tidak ada nominasi [filter label]."

---

## Step 7: Frontend — Voyage Tracking Page

File: `src/pages/customer/VoyageTrackingPage.tsx`
Route: `/customer/voyages/:nomination_id`

Fetch: `GET /api/customer/voyages/:nomination_id`.

**Install react-leaflet:**
```bash
npm install react-leaflet leaflet
npm install -D @types/leaflet
```

**Layout:**
- Left panel (1/3 width): Vessel info card (Vessel Name, Anchor Point, ETB, Estimated Duration, last AIS update timestamp)
- Right panel (2/3 width): Leaflet map centered on STS Bunati area (default center: `-3.8, 115.9`)

**Map markers:**
- Vessel position: ship icon marker at `ais_position.lat / lng`
- Anchor point zone: circle overlay at the anchor point coordinates (fetch anchor point coordinates from `/api/customer/anchor-points` if available; otherwise label only)

**Stale data warning:** If `ais_stale = true`, show yellow alert banner: "Data posisi kapal mungkin tidak terkini (terakhir diperbarui: [timestamp])."

**AIS unavailable:** If `ais_position = null`, show: "Data posisi kapal tidak tersedia saat ini."

Auto-refetch every 60 seconds.

---

## Step 8: Frontend — Weather Page

File: `src/pages/customer/WeatherPage.tsx`
Route: `/customer/weather`

Fetch: `GET /api/customer/weather` with `refetchInterval: 60000`.

**Layout:**

*Current Conditions card:*
- Large wave height display with unit (m)
- Wind speed (km/h), Visibility (km)
- Condition badge (NORMAL/WARNING/CRITICAL) with color
- "Last updated: [timestamp]"

*Alert History table:*
Columns: Level badge (WARNING/CRITICAL), Message, Time, Status (Ongoing/Resolved)
- Ongoing alerts: highlighted row background
- Resolved alerts: show resolved timestamp in Status column
- Empty state: "Tidak ada alert cuaca dalam 24 jam terakhir."

---

## Step 9: Backend — Document Master Endpoints

File: `internal/customer/document_handler.go`

### Database Migration (add before this module runs)

```sql
CREATE TABLE customer_document_master (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_master_customer_id ON customer_document_master(customer_id);
```

**No `updated_at`, no `deleted_at`. This table is append-only.**

### GET /api/customer/documents
Auth: Customer JWT.
Returns all documents for authenticated customer, ordered by `uploaded_at DESC`.

Response:
```json
[
  {
    "id": "uuid",
    "file_name": "npwp_perusahaan.pdf",
    "file_url": "...",
    "file_size_bytes": 204800,
    "mime_type": "application/pdf",
    "uploaded_at": "ISO-8601"
  }
]
```

### POST /api/customer/documents
Auth: Customer JWT. Content-Type: `multipart/form-data`.
Field: `file` (required).

Validation:
- MIME type must be `application/pdf`, `image/jpeg`, or `image/png`
- Max 10MB

Logic:
1. Save file to storage (same storage driver as M7/M8).
2. Insert row into `customer_document_master` with `customer_id` from JWT.
3. Return 201 with created document metadata.

### DELETE /api/customer/documents/:id
**Always return 403 Forbidden** with body `{ "error": "Dokumen tidak dapat dihapus." }`.
Do not implement any delete logic. Permissions are enforced at API level.

### GET /api/customer/documents/:id/download
Auth: Customer JWT.
Guard: document must belong to authenticated customer.
Redirect (302) to `file_url` or stream file directly, depending on storage driver.

---

## Step 10: Frontend — Document Master Page

File: `src/pages/customer/DocumentMasterPage.tsx`
Route: `/customer/documents`

Fetch: `GET /api/customer/documents` using TanStack React Query.

**Header:** "Document Master" title + "Upload Dokumen" button (primary)

**Upload flow:**
- "Upload Dokumen" button opens a shadcn `Dialog` with a file input.
- On file select: show file name + size preview.
- "Upload" button in dialog → `POST /api/customer/documents` (multipart).
- On success: close dialog, show toast "Dokumen berhasil diupload", refetch list.
- On error (wrong type/size): show inline error in dialog.

**Document table:**
| Column | Notes |
|--------|-------|
| File Name | Filename with file type icon |
| Type | Badge: PDF / JPG / PNG |
| Size | Human-readable (e.g. "200 KB") |
| Uploaded At | Formatted date-time |
| Actions | "Download" button only — no Edit, no Delete |

- Empty state: "Belum ada dokumen. Klik 'Upload Dokumen' untuk menambahkan."
- No delete or edit buttons anywhere on this page.

---

## Step 11: Frontend — Navigation (Updated)

File: `src/components/CustomerLayout.tsx`

Nav items (update from previous):
- Dashboard → `/customer/dashboard`
- Nominasi → `/customer/nominations`
- EPB & Invoice → `/customer/epb-invoice` ⚠️ **tambahan baru (M9b)**
- Document Master → `/customer/documents`
- Cuaca & Alert → `/customer/weather`
- Keluar (logout) → clear `customer_token` from localStorage, redirect to `/customer/login`

> EPB & Invoice nav item ditambahkan di antara "Nominasi" dan "Document Master" agar urutan mengikuti alur kerja customer: Nominasi → EPB & Invoice → Dokumen → Cuaca.

Wrap all `/customer/*` routes with `CustomerLayout` and `CustomerAuthGuard`.

---

## Acceptance Checklist
- [ ] After login, customer is redirected to /customer/dashboard
- [ ] Dashboard shows active nominations with correct status labels and color badges
- [ ] Dashboard shows active voyages (PAYMENT_CONFIRMED nominations)
- [ ] Dashboard weather widget shows current conditions with color-coded danger level
- [ ] Dashboard auto-refreshes every 60 seconds
- [ ] Nomination list shows all nominations with correct status badges
- [ ] Filter tabs (All / Active / Draft / Completed) correctly filter the list
- [ ] Voyage tracking page shows vessel position on Leaflet map
- [ ] Stale AIS data (>10 min) shows warning banner
- [ ] Weather page shows current conditions with color-coded badge
- [ ] Alert history shows last 24h alerts, distinguishes Ongoing vs Resolved
- [ ] Customer only sees their own nominations and voyages (no cross-customer data)
- [ ] "Document Master" menu item appears in sidebar navigation
- [ ] Document Master page shows uploaded documents with correct metadata
- [ ] Customer can upload new document (PDF/JPG/PNG, max 10MB) via upload dialog
- [ ] Uploaded document appears immediately in the list after upload
- [ ] No Delete or Edit buttons are visible on Document Master page
- [ ] API DELETE /api/customer/documents/:id returns 403 Forbidden
- [ ] Customer can download/preview documents from Document Master
- [ ] One customer cannot access another customer's documents
- [ ] All pages are accessible via the customer sidebar navigation
- [ ] Logout clears `customer_token` from localStorage and redirects to `/customer/login`
