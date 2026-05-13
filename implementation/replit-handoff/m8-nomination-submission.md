# Replit Handoff — M8: Nomination Request Submission

## Context
Continuation of the LPS Customer Portal. **Module 8** handles nomination creation by authenticated customers: filling the form, uploading documents, saving as Draft, and final submission to STS Platform. Requires M7 (customer auth) to be complete first.

## Prerequisites
- M7 is complete: `customers` table exists, JWT auth middleware works, `/api/auth/customer/login` returns valid JWT.
- Customer is authenticated via JWT middleware (role = `customer`).

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca dua file ini:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — foundation tokens, component library, surface preset.
2. **Per-modul UI design:** [`implementation/design/m8-nomination-submission-ui.md`](../design/m8-nomination-submission-ui.md) — form layout 5 sections, tabs dokumen, Additional Service checkbox, copy reference.

**Surface:** A — Customer Portal (Bahasa Indonesia).

**UI rules ringkas:**
- Pakai shadcn/ui basis komponen. Styling via Tailwind dari design system.
- Section Card: `rounded-2xl border border-slate-200 bg-white p-8 shadow-sm` untuk setiap section form.
- Tabs underline style untuk Dokumen Pendukung (Upload Baru / Document Master).
- File upload dropzone: lihat design system §3.2 untuk full pattern.
- Status badge: gunakan variant dari design system §2.1 (Draft = Neutral italic).
- Color primer: navy `#0F2A4D`. Canvas `bg-slate-50`. Font Inter.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate
- **File Storage:** Use local disk storage or S3-compatible bucket (`STORAGE_BUCKET` env var). Store files at `uploads/nominations/:nomination_id/:document_type`.

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_nominations.up.sql`

```sql
CREATE TABLE nominations (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id           UUID NOT NULL REFERENCES customers(id),
    nomination_number     VARCHAR(30) UNIQUE,
    vessel_name           VARCHAR(255),
    eta                   TIMESTAMPTZ,
    cargo_type            VARCHAR(255),
    cargo_quantity        DECIMAL(15,2),
    charterer             VARCHAR(255),
    estimated_barge_count INTEGER,
    nomor_pkk             VARCHAR(100),
    status                VARCHAR(40) NOT NULL DEFAULT 'DRAFT',
    sts_nomination_id     VARCHAR(100),
    submitted_at          TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
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
    nomination_id        UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    document_type        VARCHAR(50) NOT NULL,
    file_name            VARCHAR(255) NOT NULL,
    file_url             TEXT NOT NULL,
    file_size_bytes      INTEGER NOT NULL,
    document_master_id   UUID REFERENCES customer_document_master(id),
    -- NULL if uploaded directly; set if selected from Document Master
    uploaded_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nominations_customer_id ON nominations(customer_id);
CREATE INDEX idx_nominations_status      ON nominations(status);
```

Valid `document_type` values: `RENCANA_KERJA`, `SHIPPING_INSTRUCTION`, `SURAT_PENUNJUKAN_PBM`

Valid `service_key` values: `TANK_CLEANING`, `BUNKERING_FRESHWATER`, `SHORT_STAY_TEMPORARY`, `SUPPLY_LOGISTIC`, `LAY_UP`, `SHIP_CHANDLER`, `KAPAL_EMERGENCY`

---

## Step 2: Nomination Number Generator

File: `internal/nomination/number.go`

Generate nomination number format `NOM-YYYYMMDD-NNNN` (4-digit sequential per day):
```go
func GenerateNominationNumber(ctx context.Context, db *gorm.DB) (string, error) {
    today := time.Now().Format("20060102")
    prefix := "NOM-" + today + "-"
    
    var count int64
    db.WithContext(ctx).Model(&Nomination{}).
        Where("nomination_number LIKE ?", prefix+"%").
        Count(&count)
    
    return fmt.Sprintf("%s%04d", prefix, count+1), nil
}
```

---

## Step 3: Backend — Nomination Model & Repository

File: `internal/nomination/model.go`
```go
type Nomination struct {
    ID                  uuid.UUID  `gorm:"type:uuid;primaryKey"`
    CustomerID          uuid.UUID  `gorm:"not null"`
    NominationNumber    *string    `gorm:"uniqueIndex"`
    VesselName          *string
    ETA                 *time.Time
    CargoType           *string
    CargoQuantity       *float64
    Charterer           *string
    EstimatedBargeCount *int
    NomorPKK            *string
    Status              string     `gorm:"not null;default:'DRAFT'"`
    STSNominationID     *string
    SubmittedAt         *time.Time
    CreatedAt           time.Time
    UpdatedAt           time.Time
    AdditionalServices  []NominationAdditionalService `gorm:"foreignKey:NominationID"`
    Documents           []NominationDocument          `gorm:"foreignKey:NominationID"`
}

type NominationAdditionalService struct {
    ID           uuid.UUID `gorm:"type:uuid;primaryKey"`
    NominationID uuid.UUID `gorm:"not null"`
    ServiceKey   string    `gorm:"not null"`
}

type NominationDocument struct {
    ID            uuid.UUID `gorm:"type:uuid;primaryKey"`
    NominationID  uuid.UUID `gorm:"not null"`
    DocumentType  string    `gorm:"not null"`
    FileName      string    `gorm:"not null"`
    FileURL       string    `gorm:"not null"`
    FileSizeBytes int       `gorm:"not null"`
    UploadedAt    time.Time
}
```

File: `internal/nomination/repository.go`
- `Create(ctx, nomination) error`
- `FindByID(ctx, id) (*Nomination, error)`
- `FindByCustomerID(ctx, customerID) ([]Nomination, error)`
- `Update(ctx, nomination) error`
- `SetAdditionalServices(ctx, nominationID, serviceKeys []string) error` — delete existing rows then insert new ones in a transaction
- `AddDocument(ctx, doc) error`
- `FindDocumentsByNominationID(ctx, nominationID) ([]NominationDocument, error)`

---

## Step 4: Backend — Nomination Handlers

File: `internal/nomination/handler.go`

All routes require Customer JWT middleware. Extract `customer_id` from JWT context.

### POST /api/customer/nominations
Save as Draft or Submit based on `action` field in request body.

Request body:
```json
{
  "action": "draft" | "submit",
  "vessel_name": "",
  "eta": "ISO-8601",
  "cargo_type": "",
  "cargo_quantity": 0.0,
  "charterer": "",
  "estimated_barge_count": 0,
  "nomor_pkk": "",
  "additional_services": ["TANK_CLEANING", "SUPPLY_LOGISTIC"]
}
```

`additional_services` adalah array string service keys. Boleh kosong `[]` atau tidak dikirim (dianggap `[]`).

**If action = "draft":**
1. Create nomination with `status = DRAFT`, no nomination number.
2. Call `SetAdditionalServices(ctx, nominationID, req.AdditionalServices)`.
3. Return 201 with nomination ID.

**If action = "submit":**
1. Validate all fields are present and valid (ETA must be future).
2. Check all 3 document types uploaded for this nomination.
3. Check `nomor_pkk` is not empty.
4. Validate each entry in `additional_services` is a known service key; return 400 if unknown key found.
5. Call `SetAdditionalServices(ctx, nominationID, req.AdditionalServices)`.
6. Generate nomination number.
7. Set `status = SUBMITTED`, `submitted_at = now()`.
8. Async: call STS Platform API (Step 5).
9. Return 201 with nomination ID and nomination number.

### GET /api/customer/nominations
Returns all nominations for authenticated customer ordered by `created_at DESC`.
Include document list per nomination.

### GET /api/customer/nominations/:id
Return full nomination detail including documents.
Guard: nomination must belong to authenticated customer (return 403 otherwise).

### PUT /api/customer/nominations/:id
Update draft nomination. Only allowed if `status = DRAFT`.
Returns 400 if nomination is not in DRAFT status.

### POST /api/customer/nominations/:id/submit
Submit an existing draft. Same logic as submit action above.

### POST /api/customer/nominations/:id/documents
Attach a document to a nomination. Supports two modes via request body:

**Mode 1 — Upload file baru (multipart/form-data):**
Fields: `file` (required), `document_type` (required)

Validation:
- `document_type` must be one of: `RENCANA_KERJA`, `SHIPPING_INSTRUCTION`, `SURAT_PENUNJUKAN_PBM`
- File must be PDF, JPG, or PNG (check MIME type)
- File size must be ≤ 10MB
- Nomination must belong to authenticated customer
- Nomination must be in `DRAFT` or `SUBMITTED` status

Logic:
1. Save file to storage.
2. Create entry in `customer_document_master` (scoped to `customer_id`).
3. Create `nomination_documents` row with both `file_url` and `document_master_id`.

**Mode 2 — Pilih dari Document Master (JSON body):**
```json
{ "document_type": "RENCANA_KERJA", "document_master_id": "uuid" }
```

Validation:
- `document_master_id` must exist and belong to authenticated customer (return 403 otherwise).
- `document_type` must be valid.

Logic:
1. Look up `customer_document_master` by `document_master_id`.
2. Create `nomination_documents` row referencing the existing `document_master_id` (copy `file_url`, `file_name`, `file_size_bytes` from master record).
3. No new file is stored.

Both modes upsert by `document_type` (replace existing row for that type on the nomination).

---

## Step 5: Backend — STS Platform Submission

File: `internal/sts/nomination_client.go`

Called async after submit. Retry max 3× with exponential backoff (1s, 2s, 4s).

```
POST {STS_PLATFORM_BASE_URL}/api/nominations
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_nomination_id": "uuid",
  "nomination_number": "NOM-20260425-0001",
  "customer_id": "sts_customer_id_from_customers_table",
  "customer_type": "Cargo Owner",
  "vessel_name": "",
  "eta": "ISO-8601",
  "cargo_type": "",
  "cargo_quantity": 0.0,
  "cargo_unit": "MT",
  "charterer": "",
  "estimated_barge_count": 0,
  "nomor_pkk": "",
  "additional_services": ["TANK_CLEANING", "SUPPLY_LOGISTIC"],
  "documents": [
    { "type": "RENCANA_KERJA", "url": "..." },
    { "type": "SHIPPING_INSTRUCTION", "url": "..." },
    { "type": "SURAT_PENUNJUKAN_PBM", "url": "..." }
  ],
  "submitted_at": "ISO-8601"
}
```

`additional_services`: array service keys. Kirim `[]` jika tidak ada yang dipilih.

On success: update `nominations.status = PENDING`, store `sts_nomination_id` from response.
On all retries fail: update `nominations.status = SUBMIT_FAILED`; log error.

If `STS_PLATFORM_BASE_URL` is not set, set status directly to `PENDING` (mock mode for development).

---

## Step 6: Frontend — Nomination Form Page

File: `src/pages/customer/NominationFormPage.tsx`
Routes: `/customer/nominations/new` (create) and `/customer/nominations/:id/edit` (edit draft)

Use shadcn/ui `Form` with `react-hook-form` + `zod` for validation.

**Fields:**
- Vessel Name: text input, required on submit
- ETA: date-time picker, required on submit, must be future date
- Cargo Type: text input, required on submit
- Cargo Quantity (MT): number input, positive, required on submit
- Charterer: text input, required on submit
- Estimasi Jumlah Barge: integer input, min 1, required on submit
- Nomor PKK: text input, required on submit

**Additional Service section** (below main fields, above documents):

Render as a labeled checkbox group titled "Additional Service (Opsional)". Each checkbox maps service key → label:
```
TANK_CLEANING              → Tank Cleaning
BUNKERING_FRESHWATER       → Pengisian Bahan Bakar atau Air Bersih (Bunkering & Fresh Water Supplying)
SHORT_STAY_TEMPORARY       → Short Stay Temporary
SUPPLY_LOGISTIC            → Supply Logistic
LAY_UP                     → Lay Up
SHIP_CHANDLER              → Ship Chandler
KAPAL_EMERGENCY            → Kapal Emergency
```
- All unchecked by default on new form.
- Preserve checked state when re-loading a Draft (fetch from `nomination.additional_services`).
- No validation required — field is fully optional.
- On "Simpan Draft" or "Submit Nominasi": include `additional_services: string[]` in the request body (checked keys only; send `[]` if none checked).

**Document upload section** (one card per document type):
- Rencana Kerja
- Shipping Instruction  
- Surat Penunjukan PBM

Each card has two tabs:

**Tab "Upload Baru"** (default):
- File picker button (PDF/JPG/PNG, max 10MB).
- On file select: `POST /api/customer/nominations/:id/documents` (multipart mode) after nomination ID exists.
- On success: show file name + size with green checkmark; file is auto-saved to customer's Document Master.

**Tab "Pilih dari Document Master"**:
- Button "Pilih dari Document Master" → opens shadcn `Dialog` modal.
- Modal shows table of customer's Document Master (`GET /api/customer/documents`): File Name, Type, Size, Uploaded At, "Pilih" button.
- Modal has a search input to filter by filename.
- On "Pilih": close modal → `POST /api/customer/nominations/:id/documents` (JSON mode with `document_master_id`) → show selected file name with green checkmark.

Use `POST /api/customer/nominations/:id/documents` for both modes after nomination is created/saved.

**Two action buttons:**
- "Simpan Draft" → save without full validation, navigate to `/customer/nominations`
- "Submit Nominasi" → full validation + submit

**On new form:**
1. On first "Simpan Draft" click: `POST /api/customer/nominations` with `action: "draft"` → receive nomination ID → subsequent saves use `PUT /api/customer/nominations/:id`.
2. Documents can be uploaded once nomination ID exists.

**Error states:**
- Missing required fields on submit: inline field errors
- STS submission failure: toast error "Gagal mengirim nominasi. Silakan coba lagi." with retry button

---

## Step 7: Frontend — Nomination Submitted Confirmation

After successful submit:
- Navigate to `/customer/nominations` (nomination list page, handled in M10).
- Show success toast: "Nominasi [NOM-XXXXXX] berhasil disubmit dan sedang diproses oleh STS Platform."

---

## Acceptance Checklist
- [ ] Customer can create a new nomination form at /customer/nominations/new
- [ ] Customer can save form as Draft at any point without validation errors
- [ ] Draft appears in nomination list with status "Draft"
- [ ] Customer can re-open and edit draft
- [ ] Additional Service section shows 7 checkboxes with correct labels
- [ ] Customer can check zero, one, or multiple Additional Services
- [ ] Additional Service selection is preserved when re-opening a Draft
- [ ] Additional Service selection is included in STS Platform payload as `additional_services` array
- [ ] If no Additional Service selected, `additional_services: []` is sent (not null)
- [ ] Each document card shows two tabs: "Upload Baru" and "Pilih dari Document Master"
- [ ] "Upload Baru": file picker works; PDF/JPG/PNG enforced; max 10MB enforced
- [ ] "Upload Baru": on success, file is saved to Document Master AND attached to nomination
- [ ] "Pilih dari Document Master": opens modal with customer's document list and search
- [ ] "Pilih dari Document Master": selecting a document attaches it to the nomination without duplicating the file
- [ ] Invalid file type shows error message
- [ ] File > 10MB shows error message
- [ ] Submit requires all fields filled and all 3 documents attached (either method)
- [ ] Missing field on submit shows inline error
- [ ] Successful submit generates Nomination Number (NOM-YYYYMMDD-NNNN)
- [ ] After submit, customer is redirected to nomination list with success toast
- [ ] Submitted nomination cannot be edited
- [ ] Nomination data is sent to STS Platform API
- [ ] STS API failure sets status to SUBMIT_FAILED and shows retry option
