# M7 — Customer Authentication & Onboarding: Technical Specifications

## 1. Registration (FR-CA-01 to FR-CA-04)

### Form Fields & Validation — Company Information

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| Customer Code | String | No | Auto-generated on approval; read-only on form |
| Customer Name | String | Yes | Max 255 chars |
| Type | Checkbox (multi) | Yes | Min 1 selection; values: Cargo Owner, Shipper, PBM, Agen, Surveyor, Vendor Dozer |
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

## 2. Admin Validation (FR-CA-05)

### Admin View
- List customer dengan `status = PENDING_VALIDATION`.
- Kolom: Customer Name, Type, NPWP, PIC Name, Email, Phone Number, Created At.
- Admin dapat mengunduh dan mereview dokumen yang diupload.

### Actions
| Action | Status Transition | Side Effect |
|--------|------------------|-------------|
| Activate | PENDING_VALIDATION → ACTIVE | Generate Customer Code; trigger STS Platform sync; kirim email aktivasi ke customer |
| Reject | PENDING_VALIDATION → REJECTED | Catat alasan penolakan (opsional); kirim email penolakan ke customer |

### Customer Code Generation
- Format: auto-generated oleh sistem (e.g., `CUST-YYYYMM-XXXXX`).
- Digenerate saat Admin melakukan Activate.
- Unik dan tidak dapat diubah setelah digenerate.

### STS Platform Sync on Activation
- `POST /sts-api/customers` dengan payload:
  ```json
  {
    "customer_code": "...",
    "customer_name": "...",
    "type": ["Cargo Owner", "Shipper"],
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
- Token disimpan di `httpOnly` cookie (preferred) atau `localStorage`.
- Semua protected route `/customer/*` memvalidasi JWT dan cek `role = customer`.

## 4. Database Schema

```sql
CREATE TABLE customers (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_code     VARCHAR(50) UNIQUE,          -- generated on Admin activation
    customer_name     VARCHAR(255) NOT NULL,
    type              TEXT[]       NOT NULL,        -- e.g. ['Cargo Owner', 'Shipper']
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
    activated_by      UUID,  -- FK → lps_users.id (Admin who approved)
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    doc_type        VARCHAR(50) NOT NULL, -- 'NPWP' | 'NIP' | 'Company Profile'
    file_url        TEXT NOT NULL,
    file_name       VARCHAR(255),
    description     TEXT,
    issue_date      DATE,
    expiry_date     DATE,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 5. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /api/auth/customer/register | Submit registration with documents | Public |
| POST | /api/auth/customer/login | Login | Public |
| POST | /api/auth/customer/logout | Logout | Customer JWT |
| GET | /api/admin/customers/pending | List pending registrations | Admin JWT |
| GET | /api/admin/customers/:id | Detail registrasi + dokumen | Admin JWT |
| PUT | /api/admin/customers/:id/activate | Activate account + generate code | Admin JWT |
| PUT | /api/admin/customers/:id/reject | Reject account | Admin JWT |

## 6. Frontend Routes

| Route | Component | Auth Guard |
|-------|-----------|-----------|
| /register | CustomerRegisterPage | Public |
| /login | CustomerLoginPage | Public (redirect if logged in) |
| /register/success | RegistrationSuccessPage | Public |
| /customer/* | (all customer pages) | Customer JWT required |
