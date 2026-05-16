# M8 — Nomination Request Submission: Technical Specifications

> **v1.1 (Mei 2026):** Field **Vessel** tidak lagi free-text. Vessel dipilih dari **Mother Vessel (MV) milik customer ber-status `ACTIVE`** via modul M13. Sumber: `GET /api/customer/mother-vessels/active` (bukan lagi `GET /api/sts/vessels` generik). Lihat §1.1 + FR-MV-09. Konsekuensi dari modul baru M13 — dual-update dengan `module/mother-vessel-master/`.

## 1. Nomination Form (FR-NS-01)

### Form Fields & Validation
| Field | Type | Validation |
|-------|------|-----------|
| Vessel (MV) | Selection (mv_asset_id) | Required. Dipilih dari MV milik customer ber-status `ACTIVE` (M13). Bukan free-text. Lihat §1.1 |
| ETA | DateTime | Required, must be future date |
| Cargo Type | String | Required, max 255 chars |
| Cargo Quantity | Decimal | Required, positive number, unit: MT |
| Charterer | String | Required, max 255 chars |
| Estimasi Jumlah Barge | Integer | Required, min 1 |

### 1.1 Vessel Selection (v1.1 — terintegrasi M13)

- Field Vessel di form M8 adalah **selector**, bukan free-text. Customer memilih dari daftar **Mother Vessel miliknya yang ber-status `ACTIVE`**.
- Sumber data: **`GET /api/customer/mother-vessels/active`** (endpoint LPS internal, disiapkan di M13 spec §4 / handoff Step 3.5). Mengembalikan ringkas: `[{ asset_id, asset_code, vessel_name, imo_number, capacity_dwt }]`.
- MV ber-status `PENDING_APPROVAL`, `INACTIVE`, atau `REJECTED` **tidak muncul** di selector (FR-MV-09 — validasi di server-side endpoint `/active`).
- Saat customer pilih MV: simpan `mv_asset_id` (referensi ke master MV STS). `vessel_name` + `imo_number` di-snapshot dari pilihan untuk display & payload STS (denormalized — master tetap di STS).
- **Empty state:** jika customer belum punya MV `ACTIVE`, selector menampilkan pesan + link ke `/customer/mother-vessel` (menu Mother Vessel M13) untuk mendaftarkan MV lebih dulu.
- **Server-side validation (submit):** endpoint submit M8 wajib re-validate `mv_asset_id` adalah MV milik customer (owner match JWT org) ber-status `ACTIVE`. Reject `422` jika invalid (defensive — UI sudah memfilter, ini guard tambahan).
- **Dependency:** M13 (Mother Vessel Master) harus tersedia. Lihat [`module/mother-vessel-master/`](../mother-vessel-master/).

### Document Uploads (FR-NS-02)
| Document | Type | Validation |
|----------|------|-----------|
| Rencana Kerja | File | Required on Submit (optional on Draft), PDF/JPG/PNG, max 10MB |
| Shipping Instruction | File | Required on Submit, PDF/JPG/PNG, max 10MB |
| Surat Penunjukan PBM | File | Required on Submit, PDF/JPG/PNG, max 10MB |
| Nomor PKK | String (manual input) | Required on Submit, max 100 chars |

Note: Documents are optional when saving as Draft. All documents required for final Submit.

## 2. Additional Service (FR-NS-03)

Customer dapat memilih satu atau lebih Additional Service, atau tidak memilih sama sekali (opsional).

| Service Key | Label |
|-------------|-------|
| `TANK_CLEANING` | Tank Cleaning |
| `BUNKERING_FRESHWATER` | Pengisian Bahan Bakar atau Air Bersih (Bunkering & Fresh Water Supplying) |
| `SHORT_STAY_TEMPORARY` | Short Stay Temporary |
| `SUPPLY_LOGISTIC` | Supply Logistic |
| `LAY_UP` | Lay Up |
| `SHIP_CHANDLER` | Ship Chandler |
| `KAPAL_EMERGENCY` | Kapal Emergency |

- Pilihan ditampilkan sebagai checkbox multi-select di form nominasi.
- Boleh dikosongkan (0 pilihan) atau dipilih lebih dari satu.
- Tersimpan di tabel `nomination_additional_services` (many-to-many via nomination_id).
- Ikut disertakan dalam payload ke STS Platform saat submit.

## 3. Draft Behavior (FR-NS-04)
- Customer can save at any point with any subset of fields filled.
- Draft is stored with `status = DRAFT`.
- Draft is visible in the Nomination Status page (M10 FR-CD-02).
- Customer can re-open and edit draft multiple times.
- Draft is NOT sent to STS Platform.

## 4. Submit Flow (FR-NS-05, FR-NS-06)

### Pre-submit Validation
1. All form fields must be filled.
2. All required documents must be uploaded.
3. On validation fail: show field-level errors.

### On Submit
1. Validate all fields and documents.
2. Generate `nomination_number`: format `NOM-YYYYMMDD-NNNN` (sequential per day).
3. Set `status = SUBMITTED`.
4. Send nomination to STS Platform API (see payload below).
5. On STS success: set `status = PENDING` (awaiting STS processing).
6. On STS failure: set `status = SUBMIT_FAILED`; notify customer to retry; retry max 3× with exponential backoff (1s, 2s, 4s).
7. Redirect customer to Nomination Status page.

### STS API Payload (POST /sts-api/nominations)
```json
{
  "lps_nomination_id": "uuid",
  "nomination_number": "NOM-20260425-0001",
  "customer_id": "sts_customer_id",
  "vessel_name": "MV Example",
  "eta": "2026-05-01T08:00:00Z",
  "cargo_type": "Coal",
  "cargo_quantity": 50000.00,
  "cargo_unit": "MT",
  "charterer": "PT Example",
  "estimated_barge_count": 3,
  "nomor_pkk": "PKK-2026-001",
  "additional_services": ["TANK_CLEANING", "SUPPLY_LOGISTIC"],
  "documents": [
    { "type": "RENCANA_KERJA", "url": "https://storage.lps.internal/..." },
    { "type": "SHIPPING_INSTRUCTION", "url": "..." },
    { "type": "SURAT_PENUNJUKAN_PBM", "url": "..." }
  ],
  "submitted_at": "2026-04-25T10:00:00Z"
}
```

`additional_services` adalah array string (service keys). Kirim array kosong `[]` jika tidak ada pilihan.

## 5. Nomination Status Lifecycle (this module scope)
```
DRAFT → SUBMITTED → PENDING (after STS ACK)
                  ↘ SUBMIT_FAILED (STS call failed after retries)
```
Statuses beyond PENDING are handled by M9.

## 6. Database Schema

```sql
CREATE TABLE nominations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    nomination_number   VARCHAR(30) UNIQUE,
    mv_asset_id         VARCHAR(100),   -- v1.1: ref ke master MV STS (dipilih dari M13 active MV). NULL hanya untuk draft legacy.
    vessel_name         VARCHAR(255),   -- snapshot dari MV terpilih (denormalized untuk display & payload STS)
    vessel_imo          VARCHAR(20),    -- v1.1: snapshot IMO dari MV terpilih
    eta                 TIMESTAMPTZ,
    cargo_type          VARCHAR(255),
    cargo_quantity      DECIMAL(15,2),
    charterer           VARCHAR(255),
    estimated_barge_count INTEGER,
    nomor_pkk           VARCHAR(100),
    status              VARCHAR(30) NOT NULL DEFAULT 'DRAFT',
    -- DRAFT | SUBMITTED | PENDING | SUBMIT_FAILED
    -- M9 extends: APPROVED | NEED_REVISION | WAITING_PAYMENT_VERIFICATION
    --             | PAYMENT_CONFIRMED | PAYMENT_REJECTED
    sts_nomination_id   VARCHAR(100),
    submitted_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nomination_additional_services (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id  UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    service_key    VARCHAR(50) NOT NULL,
    -- TANK_CLEANING | BUNKERING_FRESHWATER | SHORT_STAY_TEMPORARY
    -- SUPPLY_LOGISTIC | LAY_UP | SHIP_CHANDLER | KAPAL_EMERGENCY
    UNIQUE (nomination_id, service_key)
);

CREATE TABLE nomination_documents (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id        UUID NOT NULL REFERENCES nominations(id),
    document_type        VARCHAR(50) NOT NULL,
    -- RENCANA_KERJA | SHIPPING_INSTRUCTION | SURAT_PENUNJUKAN_PBM
    file_name            VARCHAR(255) NOT NULL,
    file_url             TEXT NOT NULL,
    file_size_bytes      INTEGER NOT NULL,
    document_master_id   UUID REFERENCES customer_document_master(id),
    -- NULL if uploaded directly; set if selected from Document Master
    uploaded_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 7. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/nominations | List all nominations for authenticated customer | Customer JWT |
| POST | /api/customer/nominations | Create new nomination (Draft or Submit) | Customer JWT |
| GET | /api/customer/nominations/:id | Get nomination detail | Customer JWT |
| PUT | /api/customer/nominations/:id | Update draft nomination | Customer JWT |
| POST | /api/customer/nominations/:id/submit | Submit draft nomination | Customer JWT |
| POST | /api/customer/nominations/:id/documents | Upload document to nomination | Customer JWT |

## 8. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/nominations/new | NominationFormPage | New nomination form |
| /customer/nominations/:id/edit | NominationFormPage | Edit draft |
