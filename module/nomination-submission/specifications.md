# M8 — Nomination Request Submission: Technical Specifications

## 1. Nomination Form (FR-NS-01)

### Form Fields & Validation
| Field | Type | Validation |
|-------|------|-----------|
| Vessel Name | String | Required, max 255 chars |
| ETA | DateTime | Required, must be future date |
| Cargo Type | String | Required, max 255 chars |
| Cargo Quantity | Decimal | Required, positive number, unit: MT |
| Charterer | String | Required, max 255 chars |
| Estimasi Jumlah Barge | Integer | Required, min 1 |

### Document Uploads (FR-NS-02)
| Document | Type | Validation |
|----------|------|-----------|
| Rencana Kerja | File | Required on Submit (optional on Draft), PDF/JPG/PNG, max 10MB |
| Shipping Instruction | File | Required on Submit, PDF/JPG/PNG, max 10MB |
| Surat Penunjukan PBM | File | Required on Submit, PDF/JPG/PNG, max 10MB |
| Nomor PKK | String (manual input) | Required on Submit, max 100 chars |

Note: Documents are optional when saving as Draft. All documents required for final Submit.

## 2. Draft Behavior (FR-NS-03)
- Customer can save at any point with any subset of fields filled.
- Draft is stored with `status = DRAFT`.
- Draft is visible in the Nomination Status page (M10 FR-CD-02).
- Customer can re-open and edit draft multiple times.
- Draft is NOT sent to STS Platform.

## 3. Submit Flow (FR-NS-04, FR-NS-05)

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
  "documents": [
    { "type": "RENCANA_KERJA", "url": "https://storage.lps.internal/..." },
    { "type": "SHIPPING_INSTRUCTION", "url": "..." },
    { "type": "SURAT_PENUNJUKAN_PBM", "url": "..." }
  ],
  "submitted_at": "2026-04-25T10:00:00Z"
}
```

## 4. Nomination Status Lifecycle (this module scope)
```
DRAFT → SUBMITTED → PENDING (after STS ACK)
                  ↘ SUBMIT_FAILED (STS call failed after retries)
```
Statuses beyond PENDING are handled by M9.

## 5. Database Schema

```sql
CREATE TABLE nominations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    nomination_number   VARCHAR(30) UNIQUE,
    vessel_name         VARCHAR(255),
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

## 6. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/nominations | List all nominations for authenticated customer | Customer JWT |
| POST | /api/customer/nominations | Create new nomination (Draft or Submit) | Customer JWT |
| GET | /api/customer/nominations/:id | Get nomination detail | Customer JWT |
| PUT | /api/customer/nominations/:id | Update draft nomination | Customer JWT |
| POST | /api/customer/nominations/:id/submit | Submit draft nomination | Customer JWT |
| POST | /api/customer/nominations/:id/documents | Upload document to nomination | Customer JWT |

## 7. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/nominations/new | NominationFormPage | New nomination form |
| /customer/nominations/:id/edit | NominationFormPage | Edit draft |
