# Replit Handoff — M7: Customer Authentication & Onboarding

## Context
You are building the **LPS (Local Port System) Platform** — a Vessel Traffic System for PT. TBK's STS Bunati port area. This handoff covers **Module 7: Customer Authentication & Onboarding**, which is the entry point for the Customer Portal.

Customers are companies (Cargo Owner, Shipper, PBM, Agen, Surveyor, Vendor Dozer) who need to register with company documents, get admin approval, and log in before they can submit vessel nominations.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter (routing), TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo framework, GORM ORM, zerolog logger, config)
- **Database:** PostgreSQL, golang-migrate
- **File Storage:** Local disk or S3-compatible (configurable via `STORAGE_DRIVER` env: `local` | `s3`)

---

## Step 1: Database Migrations

### Migration 1 — `migrations/XXXXXX_create_customers.up.sql`

```sql
CREATE TABLE customers (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_code     VARCHAR(50) UNIQUE,          -- generated on Admin activation
    customer_name     VARCHAR(255) NOT NULL,
    type              VARCHAR(50)  NOT NULL DEFAULT 'Cargo Owner', -- always 'Cargo Owner', set by system
    npwp              VARCHAR(20)  NOT NULL UNIQUE,
    pic_name          VARCHAR(255) NOT NULL,
    email             VARCHAR(255) NOT NULL UNIQUE,
    phone             VARCHAR(20)  NOT NULL,
    address           TEXT,
    note              TEXT,
    password_hash     VARCHAR(255) NOT NULL,
    status            VARCHAR(30)  NOT NULL DEFAULT 'PENDING_VALIDATION',
    -- PENDING_VALIDATION | ACTIVE | REJECTED
    sts_customer_id   VARCHAR(100),
    rejection_reason  TEXT,
    activated_at      TIMESTAMPTZ,
    activated_by      UUID,  -- FK → lps_users.id
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_email  ON customers(email);
CREATE INDEX idx_customers_status ON customers(status);
CREATE INDEX idx_customers_npwp   ON customers(npwp);
```

### Migration 2 — `migrations/XXXXXX_create_customer_documents.up.sql`

```sql
CREATE TABLE customer_documents (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id  UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    doc_type     VARCHAR(50) NOT NULL, -- 'NPWP' | 'NIP' | 'Company Profile'
    file_url     TEXT NOT NULL,
    file_name    VARCHAR(255),
    description  TEXT,
    issue_date   DATE,
    expiry_date  DATE,
    uploaded_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customer_documents_customer_id ON customer_documents(customer_id);
```

Run: `golang-migrate -path migrations -database $DATABASE_URL up`

---

## Step 2: Backend — Models (Go)

File: `internal/customer/model.go`

```go
type Customer struct {
    ID              uuid.UUID  `gorm:"type:uuid;primaryKey"`
    CustomerCode    *string    `gorm:"uniqueIndex"`
    CustomerName    string     `gorm:"not null"`
    Type            string     `gorm:"not null;default:'Cargo Owner'"` // always "Cargo Owner", set by system
    NPWP            string     `gorm:"uniqueIndex;not null"`
    PICName         string         `gorm:"not null"`
    Email           string         `gorm:"uniqueIndex;not null"`
    Phone           string         `gorm:"not null"`
    Address         *string
    Note            *string
    PasswordHash    string         `gorm:"not null"`
    Status          string         `gorm:"not null;default:'PENDING_VALIDATION'"`
    STSCustomerID   *string
    RejectionReason *string
    ActivatedAt     *time.Time
    ActivatedBy     *uuid.UUID
    CreatedAt       time.Time
    UpdatedAt       time.Time
    Documents       []CustomerDocument `gorm:"foreignKey:CustomerID"`
}

type CustomerDocument struct {
    ID          uuid.UUID  `gorm:"type:uuid;primaryKey"`
    CustomerID  uuid.UUID  `gorm:"type:uuid;not null"`
    DocType     string     `gorm:"not null"` // NPWP | NIP | Company Profile
    FileURL     string     `gorm:"not null"`
    FileName    *string
    Description *string
    IssueDate   *time.Time
    ExpiryDate  *time.Time
    UploadedAt  time.Time
}
```

---

## Step 3: Backend — Repository

File: `internal/customer/repository.go`

```go
type Repository interface {
    Create(ctx context.Context, customer *Customer, documents []CustomerDocument) error
    FindByEmail(ctx context.Context, email string) (*Customer, error)
    FindByID(ctx context.Context, id uuid.UUID) (*Customer, error)
    FindPending(ctx context.Context) ([]Customer, error)
    FindDocuments(ctx context.Context, customerID uuid.UUID) ([]CustomerDocument, error)
    UpdateStatus(ctx context.Context, id uuid.UUID, status string, activatedBy *uuid.UUID, rejectionReason *string) error
    SetCustomerCode(ctx context.Context, id uuid.UUID, code string) error
    UpdateSTSCustomerID(ctx context.Context, id uuid.UUID, stsID string) error
}
```

`Create` must be transactional: insert `customers` row + all `customer_documents` rows in one DB transaction.

---

## Step 4: Backend — File Upload Handler

File: `internal/storage/storage.go`

Implement `Save(file multipart.File, filename string) (url string, err error)`.
- If `STORAGE_DRIVER=local`: save to `./uploads/customer-docs/` and return relative URL.
- If `STORAGE_DRIVER=s3`: upload to S3 bucket configured via `S3_BUCKET`, `S3_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.

Allowed MIME types: `application/pdf`, `image/jpeg`, `image/png`. Max size: 10MB per file.

---

## Step 5: Backend — Registration Handler

File: `internal/customer/handler.go`

### POST /api/auth/customer/register
Content-Type: `multipart/form-data`

**Form fields:**
| Field | Required | Validation |
|-------|----------|------------|
| customer_name | Yes | max 255 chars |
| npwp | Yes | regex: `^\d{2}\.\d{3}\.\d{3}\.\d-\d{3}\.\d{3}$`, unique |
| pic_name | Yes | max 255 chars |
| email | Yes | valid email, unique |
| phone | Yes | format: starts with +62 or 08 |
| address | No | max 1000 chars |
| note | No | max 2000 chars |
| password | Yes | min 8 chars |
| doc_npwp_file | Yes | file; PDF/JPG/PNG; max 10MB |
| doc_npwp_description | No | string |
| doc_npwp_issue_date | No | date string YYYY-MM-DD |
| doc_npwp_expiry_date | No | date string YYYY-MM-DD |
| doc_nip_file | Yes | file; PDF/JPG/PNG; max 10MB |
| doc_nip_description | No | string |
| doc_nip_issue_date | No | date string YYYY-MM-DD |
| doc_nip_expiry_date | No | date string YYYY-MM-DD |
| doc_company_profile_file | Yes | file; PDF/JPG/PNG; max 10MB |
| doc_company_profile_description | No | string |
| doc_company_profile_issue_date | No | date string YYYY-MM-DD |
| doc_company_profile_expiry_date | No | date string YYYY-MM-DD |

**Logic:**
1. Parse and validate all form fields.
2. Check unique email and NPWP; return 409 with field-level error if duplicate.
3. Hash password: `bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)`.
4. Save all 3 uploaded files via storage handler → get file URLs.
5. Create customer record with `status = PENDING_VALIDATION`, `type = "Cargo Owner"` (hardcoded, not from request) + 3 `customer_documents` rows (in one transaction).
6. Notify admin (in-app notification or log) of new pending registration.
7. Return 201: `{ "message": "Registrasi berhasil. Akun Anda sedang menunggu validasi Admin." }`.

---

## Step 6: Backend — Login Handler

### POST /api/auth/customer/login
Request body: `{ "email": "", "password": "" }`

Logic:
1. Find customer by email.
2. `bcrypt.CompareHashAndPassword` — mismatch → 401 `{ "error": "Email atau password salah" }`.
3. Status checks:
   - `PENDING_VALIDATION` → 403 `{ "error": "Akun menunggu validasi Admin" }`
   - `REJECTED` → 403 `{ "error": "Akun ditolak. Hubungi Admin." }`
   - `ACTIVE` → generate JWT and return 200.
4. JWT payload: `{ "customer_id": uuid, "customer_code": string, "customer_name": string, "email": string, "role": "customer", "exp": now+24h }`.
5. Sign JWT with `HS256` using `JWT_SECRET` from env.

### POST /api/auth/customer/logout
Return 200 OK (token invalidation is client-side).

---

## Step 7: Backend — Admin Handlers

File: `internal/admin/customer_handler.go`

All routes require Admin JWT (`role = admin` or `superadmin`).

### GET /api/admin/customers/pending
Returns `[]Customer` (with Documents preloaded) where `status = PENDING_VALIDATION`, ordered `created_at ASC`.

Response shape per item:
```json
{
  "id": "uuid",
  "customer_name": "",
  "type": ["Cargo Owner"],
  "npwp": "",
  "pic_name": "",
  "email": "",
  "phone": "",
  "address": "",
  "created_at": "",
  "documents": [
    { "doc_type": "NPWP", "file_url": "", "file_name": "", "description": "", "issue_date": "", "expiry_date": "" }
  ]
}
```

### GET /api/admin/customers/:id
Returns full customer detail + documents. Used for admin review before activate/reject.

### PUT /api/admin/customers/:id/activate
Logic:
1. Find customer; return 404 if not found or not PENDING_VALIDATION.
2. Generate `customer_code` — format: `CUST-YYYYMM-XXXXX` (5-digit zero-padded sequence per month, e.g. `CUST-202605-00001`). Ensure uniqueness.
3. Update `status = ACTIVE`, `customer_code`, `activated_at = now()`, `activated_by = admin_id`.
4. Async: call STS Platform sync (Step 8).
5. Async: send activation email to customer including their `customer_code`.
6. Log action to audit log.
7. Return 200 `{ "message": "Akun customer berhasil diaktifkan.", "customer_code": "CUST-202605-00001" }`.

### PUT /api/admin/customers/:id/reject
Request body: `{ "reason": "" }` (optional)
Logic:
1. Update `status = REJECTED`, `rejection_reason = reason`.
2. Async: send rejection email to customer with reason (if provided).
3. Log action to audit log.
4. Return 200 `{ "message": "Akun customer ditolak." }`.

---

## Step 8: Backend — STS Platform Sync

File: `internal/sts/client.go`

On customer activation, make this API call (async goroutine, with retry):

```
POST {STS_PLATFORM_BASE_URL}/api/customers
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_customer_id": "uuid",
  "customer_code": "CUST-202605-00001",
  "customer_name": "",
  "type": "Cargo Owner",
  "npwp": "",
  "pic_name": "",
  "email": "",
  "phone": "",
  "address": ""
}
```

- Retry max 3× with exponential backoff (1s, 2s, 4s).
- On success: store `sts_customer_id` from response into `customers.sts_customer_id`.
- On all retries fail: log error with zerolog. Do NOT roll back activation.
- If `STS_PLATFORM_BASE_URL` is not set: skip sync and log a warning.

---

## Step 9: JWT Middleware

File: `internal/middleware/customer_auth.go`

Echo middleware that:
1. Reads `Authorization: Bearer <token>` header (or cookie `lps_token`).
2. Validates JWT signature with `JWT_SECRET`.
3. Checks `role = "customer"` in payload.
4. Attaches `customer_id` and `customer_code` to Echo context.
5. On failure: return 401 `{ "error": "Unauthorized" }`.

Apply to all `/api/customer/*` routes.

---

## Step 10: Frontend — Registration Page

File: `src/pages/customer/RegisterPage.tsx`

Route: `/register` (public)

Header:
- "← Back to Login" link top-left → `/login`
- Ship icon + "Customer Registration" title + subtitle "Register your company to access the STS Platform"

**Section 1 — Company Information** (Card with title "Company Information"):

| Field | UI Component | Notes |
|-------|-------------|-------|
| Customer Code | Input (disabled) | Placeholder: "Auto-generated on approval"; helper text: "Auto-generated on approval" |
| Customer Name * | Input | Placeholder: "Enter company name" |
| NPWP * | Input | Placeholder: "XX.XXX.XXX.X-XXX.XXX" |
| PIC Name * | Input | Placeholder: "Enter PIC name" |
| Phone Number * | Input | Placeholder: "Enter phone number" |
| Email * | Input | Placeholder: "Enter email address" |
| Address | Textarea | Placeholder: "Enter company address" |
| Note | Textarea | Placeholder: "Additional notes (optional)" |

Layout: Customer Code and Customer Name are side-by-side (2-col grid). NPWP and PIC Name are side-by-side. Phone Number and Email are side-by-side. Address and Note are each full-width.

> **Note:** Field Tipe Pelanggan tidak ditampilkan di form. Sistem otomatis menetapkan `type = "Cargo Owner"` pada saat pembuatan akun.

**Section 2 — Required Documents** (Card with title "Required Documents", subtitle "All documents below are required for registration"):

Render 3 document cards — one each for: **1. NPWP**, **2. NIP**, **3. Company Profile**.

Each document card contains:
- Document title with red asterisk (required)
- `File *` label + "Choose file" upload button (shadcn Button with upload icon). On file select: show filename.
- `Description` label + Input (placeholder: "Optional description")
- `Issue Date` and `Expiry Date` side-by-side date inputs (dd/mm/yyyy format)

Footer buttons (full-width row, right-aligned):
- "Cancel" button (outline variant) → navigate to `/login`
- "Submit Registration" button (primary, navy blue) → submit form

On submit:
- Send `multipart/form-data` to `POST /api/auth/customer/register`
- Show inline validation errors per field on 422
- Show field-level error on 409 (duplicate email / NPWP)
- On 201: navigate to `/register/success`

Use react-hook-form + zod for validation. Use shadcn/ui `Form`, `Input`, `Textarea`, `Checkbox`, `Button`, `Label`.

---

## Step 11: Frontend — Registration Success Page

File: `src/pages/customer/RegisterSuccessPage.tsx`

Route: `/register/success`

Content:
- Success icon (green checkmark)
- Title: "Registrasi Berhasil"
- Message: "Akun Anda sedang menunggu validasi Admin. Kami akan menghubungi Anda melalui email setelah akun diaktifkan."
- "Kembali ke Login" button → `/login`

---

## Step 12: Frontend — Login Page

File: `src/pages/customer/LoginPage.tsx`

Route: `/login` (redirect to `/customer/dashboard` if already logged in)

Form fields: Email, Password.
Use TanStack React Query `useMutation`.

On success: store JWT in `localStorage` as `lps_token`, navigate to `/customer/dashboard`.
On 401/403: display error message from API response.

---

## Step 13: Frontend — Auth Guard

File: `src/components/CustomerAuthGuard.tsx`

Wrap all `/customer/*` routes. On mount:
1. Read token from `localStorage`.
2. Decode JWT (check `role = "customer"` and expiry).
3. Invalid/expired: clear token, redirect to `/login`.

---

## Step 14: Frontend — Admin Pending Registrations

File: `src/pages/admin/PendingCustomersPage.tsx`

Route: `/admin/customers/pending` (Admin JWT required)

Table columns: Customer Name, Type (badges), NPWP, PIC Name, Email, Phone, Registered At, Actions.

Actions per row:
- "Detail & Dokumen" button → opens Dialog showing full customer info + document list with download links.
- "Aktivasi" button (green) → `PUT /api/admin/customers/:id/activate` → refresh list → toast "Akun berhasil diaktifkan. Customer Code: [KODE]"
- "Tolak" button (red) → opens Dialog with optional reason textarea → `PUT /api/admin/customers/:id/reject` → refresh list → toast "Akun ditolak"

---

## API Endpoints Summary

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /api/auth/customer/register | Submit registration + documents | Public |
| POST | /api/auth/customer/login | Login | Public |
| POST | /api/auth/customer/logout | Logout | Customer JWT |
| GET | /api/admin/customers/pending | List pending registrations | Admin JWT |
| GET | /api/admin/customers/:id | Detail + documents | Admin JWT |
| PUT | /api/admin/customers/:id/activate | Activate + generate code | Admin JWT |
| PUT | /api/admin/customers/:id/reject | Reject with reason | Admin JWT |

---

## Environment Variables Required

```
DATABASE_URL=
JWT_SECRET=
STORAGE_DRIVER=local         # local | s3
S3_BUCKET=                   # if STORAGE_DRIVER=s3
S3_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
STS_PLATFORM_BASE_URL=       # optional; skip sync if not set
STS_API_KEY=
```

---

## Acceptance Checklist
- [ ] Customer can register with all required fields and 3 required documents
- [ ] Customer Code field is disabled/read-only in the form with "Auto-generated on approval" placeholder
- [ ] Type field is NOT shown in the form; system sets type = "Cargo Owner" automatically on record creation
- [ ] Each document section has: File upload, Description, Issue Date, Expiry Date
- [ ] Form cannot be submitted unless all 3 document files are attached
- [ ] NPWP field rejects non-standard format
- [ ] Duplicate email shows inline error: "Email sudah terdaftar"
- [ ] Duplicate NPWP shows inline error: "NPWP sudah terdaftar"
- [ ] After successful registration, confirmation page is shown
- [ ] Customer cannot login before activation
- [ ] Admin can see pending registrations list with document download links
- [ ] Admin can review full customer detail + documents via "Detail & Dokumen"
- [ ] Admin can activate → Customer Code is generated → customer status becomes ACTIVE → STS sync attempted
- [ ] Activation email to customer includes generated Customer Code
- [ ] Admin can reject with optional reason
- [ ] ACTIVE customer can login and receives JWT (payload includes customer_code)
- [ ] JWT is validated on all /customer/* routes
- [ ] PENDING/REJECTED login attempts show correct error messages
- [ ] All admin actions are logged to audit log
