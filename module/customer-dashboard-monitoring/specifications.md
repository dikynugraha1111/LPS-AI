# M10 — Customer Dashboard & Monitoring: Technical Specifications

## 1. Customer Portal Dashboard (FR-CD-03)

### Dashboard Widgets
| Widget | Data Source | Refresh |
|--------|------------|---------|
| Active Nominations Summary | LPS DB (nominations table) | On page load + TanStack Query auto-refetch every 60s |
| Active Voyages (PAYMENT_CONFIRMED) | LPS DB + STS cache | On page load + 60s |
| Weather Widget (current conditions) | LPS Weather cache (from M5) | On page load + 60s |
| Quick Action: New Nomination | — | Static link to /customer/nominations/new |

### Active Nominations Summary
- Shows nominations where `status NOT IN ('DRAFT', 'PAYMENT_CONFIRMED')` — i.e., nominations in-progress.
- Columns: Nomination Number, Vessel Name, ETA, Status (colored badge), Last Updated.
- Clicking a row navigates to `/customer/nominations/:id`.

### Active Voyages
- Shows nominations with `status = PAYMENT_CONFIRMED` where voyage is ongoing.
- Shows: Vessel Name, AIS Position (last known lat/lng), Last Position Update timestamp.
- "Lihat Detail" links to `/customer/voyages/:id`.

### Weather Widget
- Shows: Current wave height (m), Wind speed (km/h), Visibility, Condition level (Normal/Warning/Critical).
- Source: same weather cache used by LPS operators (read-only customer view).

## 2. Nomination Status List Page (FR-CD-02)

### Endpoint: GET /api/customer/nominations
Returns all nominations for the authenticated customer, ordered by `created_at DESC`.

**Includes all statuses:** DRAFT, SUBMITTED, PENDING, APPROVED, NEED_REVISION, EPB_CONFIRMATION_SUBMITTED, PAYMENT_CONFIRMED, SUBMIT_FAILED.

> **Catatan (BRD v3.1):** Status `WAITING_PAYMENT_VERIFICATION` dan `PAYMENT_REJECTED` dihapus dari tabel `nominations`. Siklus verifikasi payment (UNPAID, PENDING_REVIEW, PAYMENT_REJECT, PAID) kini dikelola oleh M9b di tabel `epb_payments` — bukan di tabel `nominations`. Status `EPB_CONFIRMATION_SUBMITTED` menggantikan `WAITING_PAYMENT_VERIFICATION` sebagai terminal status M9 di tabel `nominations`.

### UI
- Table: Nomination Number (or "Draft" if no number), Vessel Name, ETA, Status badge, Created At, Action button.
- Filter tabs: All | Active | Draft | Completed (PAYMENT_CONFIRMED).
- "New Nomination" button → `/customer/nominations/new`.
- Clicking any row → `/customer/nominations/:id`.

> **"Active" filter:** Excludes DRAFT, PAYMENT_CONFIRMED, EPB_CONFIRMATION_SUBMITTED, dan SUBMIT_FAILED. `EPB_CONFIRMATION_SUBMITTED` dikecualikan dari "Active" karena customer tidak perlu mengambil aksi di halaman ini — aksi selanjutnya ada di menu EPB & Invoice (M9b).

## 3. Voyage Tracking (FR-CD-01)

### Eligibility
Only nominations with `status = PAYMENT_CONFIRMED` show AIS position.

### Endpoint: GET /api/customer/voyages/:nomination_id
Returns:
```json
{
  "nomination_id": "uuid",
  "nomination_number": "NOM-20260425-0001",
  "vessel_name": "MV Example",
  "anchor_point": "AP-3",
  "etb": "2026-05-02T06:00:00Z",
  "ais_position": {
    "lat": -3.8123,
    "lng": 115.9876,
    "speed_knots": 4.2,
    "heading": 145,
    "last_updated": "ISO-8601"
  },
  "voyage_status": "IN_PROGRESS"  // from STS Platform cache
}
```

### Map Display
- Display vessel position on a map (Leaflet or similar).
- Show anchor point zone overlay.
- Show last AIS update timestamp; if > 10 min stale, show warning: "Data posisi mungkin tidak terkini".

## 4. Weather & Alert View (FR-CD-04)

### Endpoint: GET /api/customer/weather
Returns current weather + last 24h alert history. Sourced from LPS weather cache.

**Response:**
```json
{
  "current": {
    "wave_height_m": 1.8,
    "wind_speed_kmh": 22,
    "visibility_km": 10,
    "condition": "NORMAL",  // NORMAL | WARNING | CRITICAL
    "updated_at": "ISO-8601"
  },
  "alerts": [
    {
      "id": "uuid",
      "level": "WARNING",
      "message": "Tinggi gelombang mendekati batas aman (2.3m)",
      "created_at": "ISO-8601",
      "resolved_at": "ISO-8601 or null"
    }
  ]
}
```

### UI
- Weather card with color-coded condition indicator (green/yellow/red).
- Alert history table: Level, Message, Time, Status (Ongoing/Resolved).

## 5. Document Master (FR-CD-05 to FR-CD-08)

### Database Schema

```sql
CREATE TABLE customer_document_master (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No updated_at or deleted_at: append-only, immutable
);

CREATE INDEX idx_doc_master_customer_id ON customer_document_master(customer_id);
```

**Permissions enforced at API level:**
- `POST /api/customer/documents` — upload new document ✅ Allowed
- `DELETE /api/customer/documents/:id` — ❌ 403 Forbidden (always)
- `PUT/PATCH /api/customer/documents/:id` — ❌ 403 Forbidden (always)

### API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/documents | List all documents for authenticated customer | Customer JWT |
| POST | /api/customer/documents | Upload new document (multipart/form-data) | Customer JWT |
| GET | /api/customer/documents/:id/download | Download/preview document (redirect to file URL) | Customer JWT |
| DELETE | /api/customer/documents/:id | Always returns 403 Forbidden | Customer JWT |

**Upload rules:** PDF/JPG/PNG, max 10MB. Nomination form document upload auto-saves to Document Master (same `customer_document_master` table, scoped to customer).

### Document Linking to Nominations

`nomination_documents` table gains a new nullable FK column `document_master_id`:

```sql
ALTER TABLE nomination_documents
ADD COLUMN document_master_id UUID REFERENCES customer_document_master(id);
```

- If customer uploads new file in nomination form: file saved to storage → entry created in `customer_document_master` → `nomination_documents` row references both `nomination_id` and `document_master_id`.
- If customer selects from Document Master: `nomination_documents` row is created referencing existing `document_master_id`; no new file is stored.

### Frontend: Document Master Page

Route: `/customer/documents`

| Element | Behavior |
|---------|----------|
| Upload button | Opens file picker; on select → `POST /api/customer/documents`; shows success toast |
| Document table | Columns: File Name, Type (badge), Size, Uploaded At, Actions |
| Actions | "Download" button only — no Edit, no Delete buttons |
| Search | Filter by filename |
| Empty state | "Belum ada dokumen. Upload dokumen pertama Anda." |

### Frontend: Dual Upload in Nomination Form

In `NominationFormPage.tsx`, each document card (Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM) shows two tabs/options:

**Tab 1 — "Upload Baru"**
- Standard file input
- On upload success: file saved to Document Master; shown as selected for this nomination

**Tab 2 — "Pilih dari Document Master"**
- Button opens a modal showing customer's Document Master list
- Modal has search/filter by filename
- Customer selects one document → modal closes → document shown as selected for this nomination card
- Selection stores `document_master_id`, not a file copy

## 6. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/dashboard | Aggregated dashboard data | Customer JWT |
| GET | /api/customer/nominations | All nominations list | Customer JWT |
| GET | /api/customer/voyages/:id | Voyage detail with AIS position | Customer JWT |
| GET | /api/customer/weather | Current weather + alert history | Customer JWT |
| GET | /api/customer/documents | List Document Master | Customer JWT |
| POST | /api/customer/documents | Upload to Document Master | Customer JWT |
| GET | /api/customer/documents/:id/download | Download document | Customer JWT |

## 7. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/dashboard | CustomerDashboardPage | Landing page post-login |
| /customer/nominations | NominationListPage | All nominations + status list |
| /customer/nominations/:id | NominationStatusPage | Status detail (from M9) |
| /customer/voyages/:id | VoyageTrackingPage | Map + AIS position |
| /customer/weather | CustomerWeatherPage | Weather + alert history |
| /customer/documents | DocumentMasterPage | Document repository (append-only) |
