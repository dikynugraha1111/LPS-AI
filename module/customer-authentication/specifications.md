# M7 — Customer Authentication & Onboarding: Technical Specifications

## 1. Registration (FR-CA-01 to FR-CA-04)

### Form Fields & Validation — Company Information

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| Customer Code | String | No | Auto-generated on approval; read-only on form |
| Customer Name | String | Yes | Max 255 chars |
| NPWP | String | Yes | Format: XX.XXX.XXX.X-XXX.XXX, unique |
| PIC Name | String | Yes | Max 255 chars |
| Phone Number | String | Yes | Format: +62XXXXXXXXXX |
| Email | String | Yes | Valid email, unique |
| Address | Text | No | Max 1000 chars |
| Note | Text | No | Optional; max 2000 chars |

### Form Fields & Validation — Required Documents

Each document section contains:

| Sub-field | Type | Required | Notes |
|-----------|------|----------|-------|
| File | File upload | Yes | PDF or image; max 10MB |
| Description | String | No | Optional description |
| Issue Date | Date | No | dd/mm/yyyy |
| Expiry Date | Date | No | dd/mm/yyyy; must be after Issue Date if filled |

**Document list (all 3 are required to submit):**
1. NPWP
2. NIP
3. Company Profile

### Registration Flow
1. Customer mengisi form Company Information dan mengupload 3 dokumen wajib.
2. Customer submit → server memvalidasi semua field dan file.
3. Validasi lolos: buat akun dengan `status = PENDING_VALIDATION`. Customer Code belum di-generate.
4. Response: redirect ke halaman konfirmasi ("Akun Anda sedang menunggu validasi Admin").
5. Admin menerima notifikasi in-app tentang registrasi baru.

## 2. Admin Customer Management (FR-CA-04 to FR-CA-09)

### 2a. Customer Approval Flow (FR-CA-04 to FR-CA-07)

#### Admin Customer Management Page
- Menampilkan daftar seluruh customer dengan filter per status: PENDING_VALIDATION, ACTIVE, REJECTED.
- Kolom: Customer Name, Type, NPWP, PIC Name, Email, Phone Number, Status, Created At.
- Admin dapat membuka halaman detail per customer.

#### Admin Customer Detail Page
- Menampilkan semua data registrasi customer (Company Information + dokumen).
- Dokumen ditampilkan dengan nama file, tipe, tanggal upload; Admin dapat preview dan download.
- Tersedia tombol **Approve** dan **Reject** hanya jika status = PENDING_VALIDATION.

#### Actions
| Action | Status Transition | Side Effect |
|--------|------------------|-------------|
| Approve | PENDING_VALIDATION → ACTIVE | Generate Customer Code; kirim email aktivasi ke customer; trigger STS Platform sync |
| Reject | PENDING_VALIDATION → REJECTED | Catat alasan penolakan (opsional); kirim email penolakan ke customer |

#### Customer Code Generation
- Format: `CUST-YYYYMM-XXXXX` (auto-generated oleh sistem).
- Digenerate saat Admin melakukan Approve.
- Unik dan tidak dapat diubah setelah digenerate.

#### Notification on Approval/Rejection
- Email dikirim ke alamat email customer yang terdaftar.
- Approve: subjek "Akun LPS Portal Anda Telah Diaktifkan" dengan Customer Code.
- Reject: subjek "Registrasi LPS Portal Anda Ditolak" dengan alasan penolakan (jika diisi Admin).

### 2b. Admin Add Customer (FR-CA-08 to FR-CA-09)

#### Add Customer Form
Admin mengisi form yang sama dengan form registrasi customer (Company Information + 3 dokumen standar), dengan perbedaan:
- **Password**: Admin mengisi password awal untuk akun customer (atau sistem generate otomatis dan dikirim via email).
- **Status**: langsung ACTIVE saat disimpan.
- **Customer Code**: langsung di-generate saat disimpan (tidak perlu approval step).
- **Custom Documents** (opsional): Admin dapat menambahkan dokumen tambahan di luar 3 dokumen standar. Setiap custom document memiliki:

| Sub-field | Type | Required | Notes |
|-----------|------|----------|-------|
| Document Name / Label | String | Yes | Nama bebas, max 100 chars (e.g., "SIUP", "Akta Pendirian") |
| File | File upload | Yes | PDF or image; max 10MB |
| Description | String | No | Opsional |
| Issue Date | Date | No | dd/mm/yyyy |
| Expiry Date | Date | No | dd/mm/yyyy |

#### Flow
1. Admin membuka halaman Customer Management → klik "Tambah Customer".
2. Admin mengisi form Company Information, upload 3 dokumen standar (wajib), dan opsional tambah custom documents.
3. Admin submit → sistem memvalidasi semua field.
4. Validasi lolos: buat akun dengan `status = ACTIVE`, `customer_code` langsung di-generate, `created_by = admin_id`.
5. Sistem trigger STS Platform sync dengan payload yang sama seperti saat Approve.
6. Notifikasi email dikirim ke customer berisi informasi akun (email + password awal).

### STS Platform Sync on Activation
- `POST /sts-api/customers` dengan payload:
  ```json
  {
    "customer_code": "...",
    "customer_name": "...",
    "type": "Cargo Owner",
    "npwp": "...",
    "pic_name": "...",
    "email": "...",
    "phone": "...",
    "address": "...",
    "lps_customer_id": "..."
  }
  ```
- On success: simpan `sts_customer_id` di record customers.
- On failure: retry max 3× dengan exponential backoff (1s, 2s, 4s). Log failure. Akun tetap ACTIVE.

## 3. Login (FR-CA-06)

### Endpoint: POST /api/auth/customer/login
**Request:** `{ email, password }`

**Response matrix:**
| Condition | HTTP | Response |
|-----------|------|----------|
| Valid credentials, ACTIVE | 200 | JWT token + customer profile |
| Valid credentials, PENDING_VALIDATION | 403 | "Akun menunggu validasi Admin" |
| Valid credentials, REJECTED | 403 | "Akun ditolak. Hubungi Admin." |
| Invalid credentials | 401 | "Email atau password salah" |

**JWT payload:** `{ customer_id, customer_code, customer_name, email, role: "customer", exp: 24h }`

### Session
- Token disimpan di `localStorage` sebagai `customer_token`.
- Semua protected route `/customer/*` memvalidasi JWT dan cek `role = customer`.

## 4. Database Schema

```sql
CREATE TABLE customers (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_code     VARCHAR(50) UNIQUE,          -- generated on Admin approval or manual add
    customer_name     VARCHAR(255) NOT NULL,
    type              VARCHAR(50)  NOT NULL DEFAULT 'Cargo Owner',
    npwp              VARCHAR(20)  NOT NULL UNIQUE,
    pic_name          VARCHAR(255) NOT NULL,
    email             VARCHAR(255) NOT NULL UNIQUE,
    phone             VARCHAR(20)  NOT NULL,
    address           TEXT,
    note              TEXT,
    password_hash     VARCHAR(255) NOT NULL,
    status            VARCHAR(30)  NOT NULL DEFAULT 'PENDING_VALIDATION',
    -- PENDING_VALIDATION | ACTIVE | REJECTED
    registration_source VARCHAR(20) NOT NULL DEFAULT 'SELF',
    -- SELF (customer self-register) | ADMIN (added by admin)
    sts_customer_id   VARCHAR(100),
    rejection_reason  TEXT,
    activated_at      TIMESTAMPTZ,
    activated_by      UUID,          -- FK → lps_users.id (Admin who approved or created)
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    doc_type        VARCHAR(50) NOT NULL,
    -- Standard types: 'NPWP' | 'NIP' | 'Company Profile'
    -- Custom types (admin-added): any string set by Admin
    doc_label       VARCHAR(100),     -- used for custom documents; NULL for standard types
    is_custom       BOOLEAN NOT NULL DEFAULT FALSE,
    file_url        TEXT NOT NULL,
    file_name       VARCHAR(255),
    description     TEXT,
    issue_date      DATE,
    expiry_date     DATE,
    uploaded_by     VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER', -- 'CUSTOMER' | 'ADMIN'
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 5. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /api/auth/customer/register | Submit registration with documents | Public |
| POST | /api/auth/customer/login | Login | Public |
| POST | /api/auth/customer/logout | Logout | Customer JWT |
| GET | /api/admin/customers | List all customers (filter by status) | Admin JWT |
| GET | /api/admin/customers/pending | List pending registrations | Admin JWT |
| GET | /api/admin/customers/:id | Detail customer + dokumen | Admin JWT |
| PUT | /api/admin/customers/:id/approve | Approve account + generate Customer Code + send notif | Admin JWT |
| PUT | /api/admin/customers/:id/reject | Reject account + send notif | Admin JWT |
| POST | /api/admin/customers | Add new customer (admin manual add) | Admin JWT |

## 6. Frontend Routes

| Route | Component | Auth Guard |
|-------|-----------|-----------|
| /register | CustomerRegisterPage | Public |
| /customer/login | CustomerLoginPage | Public (redirect to /customer/dashboard if logged in) |
| /register/success | RegistrationSuccessPage | Public |
| /customer/dashboard | CustomerDashboardPage | Customer JWT required |
| /customer/* | (all customer pages) | Customer JWT required |

> **Note:** JWT token stored in `localStorage` as `customer_token`. Admin routes are entirely separate under `/admin/*` (separate login at `/admin/login`, HTTP-only cookie auth).
