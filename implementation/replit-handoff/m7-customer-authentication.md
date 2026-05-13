# Replit Handoff — M7: Customer Authentication & Onboarding

> **Version history:**
> - **v1.0** — Initial handoff: self-registration, admin pending list + activate/reject, login, JWT middleware
> - **v1.1** — 2026-05-06: Admin Customer Management (full list + filter), Admin Add Customer manual, custom documents, rename activate→approve, notification email on approve/reject, DB schema additions (`registration_source`, `doc_label`, `is_custom`, `uploaded_by`)

---

## Context
You are building the **LPS (Local Port System) Platform** — a Vessel Traffic System for PT. TBK's STS Bunati port area. This handoff covers **Module 7: Customer Authentication & Onboarding**, which is the entry point for the Customer Portal.

Customers are companies (Cargo Owner, Shipper, PBM, Agen, Surveyor, Vendor Dozer) who need to register with company documents, get admin approval, and log in before they can submit vessel nominations.

Admin can also add customers directly from the Admin Dashboard (bypassing the self-registration flow), and may attach custom documents beyond the 3 standard ones.

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca dua file ini:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — foundation tokens (warna, typography, spacing, radius, shadow), component library, dua surface preset.
2. **Per-modul UI design:** [`implementation/design/m7-customer-authentication-ui.md`](../design/m7-customer-authentication-ui.md) — page inventory, layout per page, komponen, copy reference.

**Surface mapping M7:**
- **Surface A — Customer Portal** (Bahasa Indonesia): customer register, login, register success.
- **Surface B — Internal Operator** (English): admin login, Customer Management list, customer detail (approve/reject), Add Customer manual.

**Layout pattern per halaman auth (PENTING — dari design system v1.1):**

| Halaman | Layout | Catatan |
|---|---|---|
| `/customer/login` | **Auth Split-Screen** (design system §3.4) | Form panel kiri 50% + Image Showcase Carousel kanan 50%. Pada `<lg` image hidden. |
| `/customer/register` | **Auth Split-Screen** | Sama dengan login. Form panel scrollable (form lebih panjang), image showcase fixed. `max-w-xl` untuk form (vs `max-w-md` di login). |
| `/customer/register/success` | **Auth Centered** | Simple confirmation card. |
| `/admin/login` | **Auth Centered** | Minimal, no showcase. English copy. Token di HTTP-only cookie. |
| Halaman admin lain | Sidebar B + Top bar B layout | Standard operator layout. |

**Required komponen baru (v1.1):**
- **Input with leading icon** (design system §3.1) — wajib untuk email & password fields di semua auth pages. Icon kiri `Mail` / `Lock`, eye toggle kanan untuk password.
- **Image Showcase Carousel** (design system §3.4) — 3 default slides Surface A (lihat tabel di design system). Auto-advance 6 detik, pause on hover. Images di `public/images/auth-showcase/`, config di `src/config/authSlides.ts`.
- **Brand mark wordmark inline** (design system §2.10) — anchor icon + "LPS System" di top-left auth pages. BUKAN brand stacked.
- **Divider with text** (design system §3.1) — untuk "atau" di customer login.

**UI rules ringkas:**
- shadcn/ui sebagai basis komponen; styling via Tailwind class dari design system.
- Heading customer login: "Selamat Datang" `text-4xl font-semibold`. Subtitle: "Masuk ke Portal Pelanggan LPS".
- Heading customer register: "Daftar Akun Pelanggan". Subtitle: "Lengkapi data perusahaan dan unggah dokumen wajib."
- Heading admin login: "Sign In". Subtitle: "Operations console — Bunati Port".
- Status badge: variant dari design system §2.1 (soft pill 50/200/700 dengan border).
- Color primer: navy `#0F2A4D` (sidebar, primary button), canvas `bg-slate-50`, card `bg-white`, border `border-slate-200`.
- Card: `rounded-2xl border border-slate-200 bg-white p-8 shadow-sm`.
- Font: Inter. Heading semibold (600).
- Icon: Lucide React.
- Footer copy customer auth: "© 2026 LPS System. Seluruh hak dilindungi." (ID).
- Footer copy admin login: "© 2026 LPS System" (EN).

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter (routing), TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo framework, GORM ORM, zerolog logger, config)
- **Database:** PostgreSQL, golang-migrate
- **File Storage:** Local disk or S3-compatible (configurable via `STORAGE_DRIVER` env: `local` | `s3`)
- **Email:** SMTP via env vars (for activation/rejection/welcome notifications)

---

## Step 1: Database Migrations

### Migration 1 — `migrations/XXXXXX_create_customers.up.sql`

> ⚠️ **v1.1 change:** Tambah kolom `registration_source` (SELF / ADMIN).

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_code       VARCHAR(50) UNIQUE,          -- generated on Admin approve or manual add
    customer_name       VARCHAR(255) NOT NULL,
    type                VARCHAR(50)  NOT NULL DEFAULT 'Cargo Owner',
    npwp                VARCHAR(20)  NOT NULL UNIQUE,
    pic_name            VARCHAR(255) NOT NULL,
    email               VARCHAR(255) NOT NULL UNIQUE,
    phone               VARCHAR(20)  NOT NULL,
    address             TEXT,
    note                TEXT,
    password_hash       VARCHAR(255) NOT NULL,
    status              VARCHAR(30)  NOT NULL DEFAULT 'PENDING_VALIDATION',
    -- PENDING_VALIDATION | ACTIVE | REJECTED
    registration_source VARCHAR(20)  NOT NULL DEFAULT 'SELF',
    -- SELF = customer self-registered | ADMIN = added directly by Admin
    sts_customer_id     VARCHAR(100),
    rejection_reason    TEXT,
    activated_at        TIMESTAMPTZ,
    activated_by        UUID,  -- FK → lps_users.id
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_email               ON customers(email);
CREATE INDEX idx_customers_status              ON customers(status);
CREATE INDEX idx_customers_npwp                ON customers(npwp);
CREATE INDEX idx_customers_registration_source ON customers(registration_source);
```

### Migration 2 — `migrations/XXXXXX_create_customer_documents.up.sql`

> ⚠️ **v1.1 change:** Tambah kolom `doc_label` (untuk custom document name), `is_custom` (flag), `uploaded_by` (CUSTOMER / ADMIN).

```sql
CREATE TABLE customer_documents (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id  UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    doc_type     VARCHAR(50) NOT NULL,
    -- Standard: 'NPWP' | 'NIP' | 'Company Profile'
    -- Custom: free string set by Admin (e.g. 'SIUP', 'Akta Pendirian')
    doc_label    VARCHAR(100),          -- human-readable label for custom docs; NULL for standard
    is_custom    BOOLEAN NOT NULL DEFAULT FALSE,
    file_url     TEXT NOT NULL,
    file_name    VARCHAR(255),
    description  TEXT,
    issue_date   DATE,
    expiry_date  DATE,
    uploaded_by  VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER', -- 'CUSTOMER' | 'ADMIN'
    uploaded_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customer_documents_customer_id ON customer_documents(customer_id);
CREATE INDEX idx_customer_documents_is_custom   ON customer_documents(is_custom);
```

Run: `golang-migrate -path migrations -database $DATABASE_URL up`

---

## Step 2: Backend — Models (Go)

File: `internal/customer/model.go`

> ⚠️ **v1.1 change:** Tambah field `RegistrationSource` di `Customer`; tambah `DocLabel`, `IsCustom`, `UploadedBy` di `CustomerDocument`.

```go
type Customer struct {
    ID                 uuid.UUID  `gorm:"type:uuid;primaryKey"`
    CustomerCode       *string    `gorm:"uniqueIndex"`
    CustomerName       string     `gorm:"not null"`
    Type               string     `gorm:"not null;default:'Cargo Owner'"`
    NPWP               string     `gorm:"uniqueIndex;not null"`
    PICName            string     `gorm:"not null"`
    Email              string     `gorm:"uniqueIndex;not null"`
    Phone              string     `gorm:"not null"`
    Address            *string
    Note               *string
    PasswordHash       string     `gorm:"not null"`
    Status             string     `gorm:"not null;default:'PENDING_VALIDATION'"`
    // PENDING_VALIDATION | ACTIVE | REJECTED
    RegistrationSource string     `gorm:"not null;default:'SELF'"`
    // SELF | ADMIN
    STSCustomerID      *string
    RejectionReason    *string
    ActivatedAt        *time.Time
    ActivatedBy        *uuid.UUID
    CreatedAt          time.Time
    UpdatedAt          time.Time
    Documents          []CustomerDocument `gorm:"foreignKey:CustomerID"`
}

type CustomerDocument struct {
    ID          uuid.UUID  `gorm:"type:uuid;primaryKey"`
    CustomerID  uuid.UUID  `gorm:"type:uuid;not null"`
    DocType     string     `gorm:"not null"`
    // Standard: 'NPWP' | 'NIP' | 'Company Profile'
    // Custom: free string set by Admin
    DocLabel    *string    // human-readable label for custom docs; nil for standard
    IsCustom    bool       `gorm:"not null;default:false"`
    FileURL     string     `gorm:"not null"`
    FileName    *string
    Description *string
    IssueDate   *time.Time
    ExpiryDate  *time.Time
    UploadedBy  string     `gorm:"not null;default:'CUSTOMER'"` // CUSTOMER | ADMIN
    UploadedAt  time.Time
}
```

---

## Step 3: Backend — Repository

File: `internal/customer/repository.go`

> ⚠️ **v1.1 change:** Tambah `FindAll` (dengan filter status), `CreateByAdmin`.

```go
type Repository interface {
    // Shared
    Create(ctx context.Context, customer *Customer, documents []CustomerDocument) error
    FindByEmail(ctx context.Context, email string) (*Customer, error)
    FindByID(ctx context.Context, id uuid.UUID) (*Customer, error)
    FindDocuments(ctx context.Context, customerID uuid.UUID) ([]CustomerDocument, error)
    UpdateStatus(ctx context.Context, id uuid.UUID, status string, activatedBy *uuid.UUID, rejectionReason *string) error
    SetCustomerCode(ctx context.Context, id uuid.UUID, code string) error
    UpdateSTSCustomerID(ctx context.Context, id uuid.UUID, stsID string) error

    // v1.1: Admin Customer Management
    FindAll(ctx context.Context, statusFilter string) ([]Customer, error) // statusFilter: "" = all, or "PENDING_VALIDATION" | "ACTIVE" | "REJECTED"
    FindPending(ctx context.Context) ([]Customer, error)                  // convenience alias for FindAll("PENDING_VALIDATION")
    CreateByAdmin(ctx context.Context, customer *Customer, standardDocs []CustomerDocument, customDocs []CustomerDocument) error
}
```

`Create` dan `CreateByAdmin` harus transaksional: insert `customers` + semua `customer_documents` dalam satu DB transaction.

`CreateByAdmin`: set `registration_source = "ADMIN"`, `status = "ACTIVE"`, generate `customer_code` sebelum insert, set `uploaded_by = "ADMIN"` untuk semua dokumen. Custom docs: `is_custom = true`, `doc_type = doc_label`.

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

> ⚠️ **v1.1 change:** Tambah `GET /api/admin/customers` (full list + filter), `POST /api/admin/customers` (admin add). Rename `activate` → `approve`. Tambah email notification di approve dan reject.

### GET /api/admin/customers
Query param: `?status=` (optional; blank = all records).
Returns `[]Customer` ordered `created_at DESC`.

Response shape per item:
```json
{
  "id": "uuid",
  "customer_code": "CUST-202605-00001",
  "customer_name": "",
  "npwp": "",
  "pic_name": "",
  "email": "",
  "phone": "",
  "status": "ACTIVE",
  "registration_source": "SELF",
  "created_at": ""
}
```

### GET /api/admin/customers/pending
Returns `[]Customer` (with Documents preloaded) where `status = PENDING_VALIDATION`, ordered `created_at ASC`.

Response shape per item:
```json
{
  "id": "uuid",
  "customer_name": "",
  "npwp": "",
  "pic_name": "",
  "email": "",
  "phone": "",
  "address": "",
  "created_at": "",
  "documents": [
    {
      "doc_type": "NPWP",
      "doc_label": null,
      "is_custom": false,
      "file_url": "",
      "file_name": "",
      "description": "",
      "issue_date": "",
      "expiry_date": ""
    }
  ]
}
```

### GET /api/admin/customers/:id
Returns full customer detail + all documents (standard and custom). Used for admin review.

### PUT /api/admin/customers/:id/approve
> ⚠️ **v1.1:** Renamed dari `/activate` → `/approve`. Tambah email notification.

Logic:
1. Find customer; return 404 if not found. Return 400 if status ≠ PENDING_VALIDATION.
2. Generate `customer_code` — format: `CUST-YYYYMM-XXXXX` (5-digit zero-padded sequence per month). Ensure uniqueness via DB unique constraint retry.
3. Update `status = ACTIVE`, `customer_code`, `activated_at = now()`, `activated_by = admin_id`.
4. Async: call STS Platform sync (Step 8).
5. Async: send approval email to customer.
   - Subject: "Akun LPS Portal Anda Telah Diaktifkan"
   - Body: nama customer, Customer Code, URL login (`FRONTEND_URL/customer/login`).
6. Log action to audit log.
7. Return 200 `{ "message": "Akun customer berhasil diaktifkan.", "customer_code": "CUST-202605-00001" }`.

### PUT /api/admin/customers/:id/reject
Request body: `{ "reason": "" }` (optional)

Logic:
1. Find customer; return 404 if not found. Return 400 if status ≠ PENDING_VALIDATION.
2. Update `status = REJECTED`, `rejection_reason = reason`.
3. Async: send rejection email to customer.
   - Subject: "Registrasi LPS Portal Anda Ditolak"
   - Body: nama customer, alasan penolakan (jika diisi), instruksi untuk menghubungi Admin.
4. Log action to audit log.
5. Return 200 `{ "message": "Akun customer ditolak." }`.

### POST /api/admin/customers
> ⚠️ **v1.1:** Endpoint baru — Admin Add Customer manual.

Content-Type: `multipart/form-data`

**Standard form fields** (sama dengan registration):
| Field | Required |
|-------|----------|
| customer_name | Yes |
| npwp | Yes |
| pic_name | Yes |
| email | Yes |
| phone | Yes |
| address | No |
| note | No |
| password | Yes |
| doc_npwp_file | Yes |
| doc_npwp_description | No |
| doc_npwp_issue_date | No |
| doc_npwp_expiry_date | No |
| doc_nip_file | Yes |
| doc_nip_description | No |
| doc_nip_issue_date | No |
| doc_nip_expiry_date | No |
| doc_company_profile_file | Yes |
| doc_company_profile_description | No |
| doc_company_profile_issue_date | No |
| doc_company_profile_expiry_date | No |

**Custom document fields** (repeatable, 0 or more):
| Field | Pattern | Required |
|-------|---------|----------|
| custom_doc_label | `custom_doc_label[0]`, `custom_doc_label[1]`, … | Yes (per entry) |
| custom_doc_file | `custom_doc_file[0]`, `custom_doc_file[1]`, … | Yes (per entry) |
| custom_doc_description | `custom_doc_description[0]`, … | No |
| custom_doc_issue_date | `custom_doc_issue_date[0]`, … | No |
| custom_doc_expiry_date | `custom_doc_expiry_date[0]`, … | No |

**Logic:**
1. Validate all fields; check unique email and NPWP.
2. Hash password.
3. Save all files via storage handler.
4. Build standard `CustomerDocument` slice (3 items, `is_custom = false`, `uploaded_by = "ADMIN"`).
5. Build custom `CustomerDocument` slice from `custom_doc_*[]` arrays (`is_custom = true`, `doc_label = label`, `doc_type = label`, `uploaded_by = "ADMIN"`).
6. Generate `customer_code` immediately.
7. Call `repo.CreateByAdmin(ctx, customer, standardDocs, customDocs)` — transactional.
8. Async: STS Platform sync (same payload as approve flow).
9. Async: send welcome email to customer with login credentials.
   - Subject: "Akun LPS Portal Anda Telah Dibuat"
   - Body: nama customer, Customer Code, email, password awal, URL login.
10. Return 201 `{ "message": "Customer berhasil ditambahkan.", "customer_code": "CUST-202605-00002" }`.

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
1. Reads `Authorization: Bearer <token>` header.
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
- "← Back to Login" link top-left → `/customer/login`
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
- "Cancel" button (outline variant) → navigate to `/customer/login`
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
- "Kembali ke Login" button → `/customer/login`

---

## Step 12: Frontend — Login Page

File: `src/pages/customer/LoginPage.tsx`

Route: `/customer/login` (redirect to `/customer/dashboard` if already logged in)

Form fields: Email, Password.
Use TanStack React Query `useMutation`.

On success: store JWT in `localStorage` as `customer_token`, navigate to `/customer/dashboard`.
On 401/403: display error message from API response.

---

## Step 13: Frontend — Auth Guard

File: `src/components/CustomerAuthGuard.tsx`

Wrap all `/customer/*` routes. On mount:
1. Read `customer_token` from `localStorage`.
2. Decode JWT (check `role = "customer"` and expiry).
3. Invalid/expired: clear token, redirect to `/customer/login`.

---

## Step 14: Frontend — Admin Customer Management Page

> ⚠️ **v1.1 change:** Halaman ini menggantikan `PendingCustomersPage.tsx` yang hanya menampilkan pending. Sekarang menampilkan semua customer dengan tab/filter per status, plus tombol "Tambah Customer".

File: `src/pages/admin/CustomerManagementPage.tsx`

Route: `/admin/customers` (Admin JWT required; redirect to `/admin/login` if not authenticated)

**Layout:**
- Page title: "Customer Management"
- Tombol "Tambah Customer" di kanan atas → navigate to `/admin/customers/new`
- Tab bar atau filter dropdown: **All** | **Pending** | **Active** | **Rejected**
- Tabel dengan kolom: No, Customer Name, NPWP, PIC Name, Email, Phone, Status (badge), Source (Self / Admin), Registered At, Actions

**Status badge colors:**
- `PENDING_VALIDATION` → yellow / amber
- `ACTIVE` → green
- `REJECTED` → red

**Actions per row:**
- "Detail" button → navigate to `/admin/customers/:id`

**Customer Detail Page** (`src/pages/admin/CustomerDetailPage.tsx`, route `/admin/customers/:id`):
- Tampilkan semua data Company Information.
- Section "Documents": daftar semua dokumen. Setiap dokumen menampilkan: nama file, tipe (badge "Standard" atau "Custom"), label (untuk custom doc), tanggal upload, tombol "Download".
- Jika status = `PENDING_VALIDATION`: tampilkan tombol **"Approve"** (green) dan **"Reject"** (red).
  - Klik "Approve" → confirm dialog → `PUT /api/admin/customers/:id/approve` → redirect ke `/admin/customers` → toast "Akun berhasil diaktifkan. Customer Code: [KODE]"
  - Klik "Reject" → modal dengan textarea alasan (opsional) → `PUT /api/admin/customers/:id/reject` → redirect ke `/admin/customers` → toast "Akun ditolak"
- Jika status bukan PENDING_VALIDATION: tombol Approve/Reject tidak ditampilkan.

---

## Step 15: Frontend — Admin Add Customer Page

> ⚠️ **v1.1: Halaman baru.**

File: `src/pages/admin/AddCustomerPage.tsx`

Route: `/admin/customers/new` (Admin JWT required)

**Section 1 — Company Information** (sama dengan form registrasi customer):

| Field | Required |
|-------|----------|
| Customer Name | Yes |
| NPWP | Yes |
| PIC Name | Yes |
| Phone Number | Yes |
| Email | Yes |
| Address | No |
| Note | No |
| Password (awal) | Yes |

> Customer Code **tidak ditampilkan** di form (akan di-generate otomatis saat save).

**Section 2 — Standard Documents** (sama dengan form registrasi):
- 3 document cards wajib: NPWP, NIP, Company Profile.
- Setiap card: File upload (required), Description, Issue Date, Expiry Date.

**Section 3 — Custom Documents** (opsional):
- Default: kosong (tidak ada custom doc).
- Tombol **"+ Tambah Dokumen"** menambah satu document card baru.
- Setiap custom document card memiliki field:
  - Document Name / Label (Input, required, placeholder: "e.g. SIUP, Akta Pendirian")
  - File upload (required)
  - Description (optional)
  - Issue Date, Expiry Date (optional, side-by-side)
  - Tombol "×" untuk menghapus card tersebut
- Tidak ada batas maksimum jumlah custom doc.

**Footer buttons:**
- "Cancel" (outline) → `/admin/customers`
- "Simpan Customer" (primary) → submit form

**On submit:**
- Build `FormData`, append semua standard doc fields, loop custom docs dengan index (`custom_doc_label[0]`, `custom_doc_file[0]`, dsb.).
- `POST /api/admin/customers`
- On 201: navigate to `/admin/customers` → toast "Customer berhasil ditambahkan."
- On 409: tampilkan inline error per field (duplicate email / NPWP).

Gunakan react-hook-form + zod + `useFieldArray` untuk dynamic custom document list.

---

## API Endpoints Summary

> ⚠️ **v1.1 changes:** Tambah `GET /api/admin/customers`, `POST /api/admin/customers`. Rename `activate` → `approve`.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /api/auth/customer/register | Submit registration + documents | Public |
| POST | /api/auth/customer/login | Login | Public |
| POST | /api/auth/customer/logout | Logout | Customer JWT |
| GET | /api/admin/customers | List all customers (filter by status) | Admin JWT |
| GET | /api/admin/customers/pending | List pending registrations | Admin JWT |
| GET | /api/admin/customers/:id | Detail + documents | Admin JWT |
| PUT | /api/admin/customers/:id/approve | Approve + generate code + send email | Admin JWT |
| PUT | /api/admin/customers/:id/reject | Reject + send email | Admin JWT |
| POST | /api/admin/customers | Add customer manually (admin) | Admin JWT |

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
FRONTEND_URL=                # used in email notification links (e.g. https://lps.example.com)
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=                   # e.g. noreply@lps.example.com
```

---

## Acceptance Checklist

### v1.0 — Self-Registration & Login
- [ ] Customer can register with all required fields and 3 required documents
- [ ] Customer Code field is disabled/read-only in the form with "Auto-generated on approval" placeholder
- [ ] Type field is NOT shown in the form; system sets type = "Cargo Owner" automatically
- [ ] Each document section has: File upload, Description, Issue Date, Expiry Date
- [ ] Form cannot be submitted unless all 3 document files are attached
- [ ] NPWP field rejects non-standard format
- [ ] Duplicate email shows inline error: "Email sudah terdaftar"
- [ ] Duplicate NPWP shows inline error: "NPWP sudah terdaftar"
- [ ] After successful registration, confirmation page is shown
- [ ] Customer cannot login before activation
- [ ] ACTIVE customer can login and receives JWT (payload includes customer_code)
- [ ] JWT is validated on all /customer/* routes
- [ ] PENDING/REJECTED login attempts show correct error messages

### v1.1 — Admin Customer Management & Add Customer
- [ ] Admin Customer Management page menampilkan semua customer dengan filter tab: All / Pending / Active / Rejected
- [ ] Tabel menampilkan kolom Status (badge warna) dan Source (Self / Admin)
- [ ] Admin dapat membuka halaman Customer Detail dari tabel
- [ ] Customer Detail menampilkan semua dokumen; standard doc dan custom doc dibedakan dengan badge
- [ ] Tombol Approve dan Reject hanya muncul jika status = PENDING_VALIDATION
- [ ] Approve → Customer Code di-generate → status ACTIVE → email aktivasi terkirim ke customer (berisi Customer Code dan URL login)
- [ ] Reject → status REJECTED → email penolakan terkirim ke customer (berisi alasan jika diisi)
- [ ] Admin Add Customer: form dapat disubmit dengan 3 dokumen standar wajib
- [ ] Admin Add Customer: custom document dapat ditambah dinamis dengan tombol "+ Tambah Dokumen" dan dihapus per card
- [ ] Customer yang dibuat Admin langsung ACTIVE; Customer Code langsung di-generate
- [ ] Welcome email terkirim ke customer yang dibuat Admin (berisi email, password awal, Customer Code, URL login)
- [ ] `registration_source` tersimpan benar: SELF untuk self-register, ADMIN untuk yang dibuat Admin
- [ ] Custom documents tersimpan dengan `is_custom = true` dan `doc_label` yang benar
- [ ] STS Platform sync dipanggil untuk approve dan admin-add (tidak untuk reject)
- [ ] Semua aksi Admin (approve, reject, add customer) tercatat di audit log
