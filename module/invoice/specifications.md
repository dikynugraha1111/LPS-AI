# M9c — Invoice (Customer Portal): Technical Specifications

> **Modul baru di v3.3.**

## 1. Payment Status Lifecycle

```
[STS webhook EPB_SHORTFALL_DETECTED / ADDITIONAL_SERVICE_INVOICE]
        │
        ▼
   Belum Dibayar (UNPAID) ──► customer klik "Bayar" ──► upload proof + submit
                                                              │
                                                              ▼
                                              Menunggu Verifikasi (WAITING_PAYMENT_VERIFICATION) ──► STS verifies
                                                              │
                                                  ┌───────────┴────────────┐
                                                  ▼                        ▼
                                       Pembayaran Ditolak              Lunas (PAID)
                                       (PAYMENT_REJECT)                (view-only, terminal)
                                       (klik Revisi Data)
                                                  │
                                                  ▼
                                          Upload + Re-Submit
                                                  │
                                                  └──► Menunggu Verifikasi (loop)
```

Statuses (sama dengan EPB untuk konsistensi UX): `UNPAID`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_REJECT`, `PAID`

**Label mapping ke UI (Bahasa Indonesia):**
- `UNPAID` → "Belum Dibayar"
- `WAITING_PAYMENT_VERIFICATION` → "Menunggu Verifikasi"
- `PAYMENT_REJECT` → "Pembayaran Ditolak"
- `PAID` → "Lunas"

> Tidak ada partial payment di Invoice — harus dibayar penuh. Berbeda dengan EPB.
> Invoice tidak mengubah `nominations.status`. Invoice = settlement separate.

## 2. Database Schema

```sql
-- Core Invoice table
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number      VARCHAR(40) NOT NULL UNIQUE,    -- format: INV-YYYYMMDD-NNNNN (dari STS)
    customer_id         UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    nomination_id       UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    source              VARCHAR(30) NOT NULL,           -- 'EPB_SHORTFALL' | 'ADDITIONAL_SERVICE'
    parent_epb_id       UUID REFERENCES epb_payments(id),  -- NOT NULL jika source='EPB_SHORTFALL'
    service_keys        TEXT[],                          -- NOT NULL jika source='ADDITIONAL_SERVICE' (mis. ['tank_cleaning','bunkering'])
    amount              NUMERIC(18,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'IDR',
    due_date            TIMESTAMPTZ,
    status              VARCHAR(40) NOT NULL DEFAULT 'UNPAID',
    -- 'UNPAID' | 'WAITING_PAYMENT_VERIFICATION' | 'PAYMENT_REJECT' | 'PAID'
    rejection_reason    TEXT,
    confirmed_at        TIMESTAMPTZ,
    sts_idempotency_key VARCHAR(255) UNIQUE,             -- guard untuk duplicate webhook create
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT invoices_source_check CHECK (
        (source = 'EPB_SHORTFALL' AND parent_epb_id IS NOT NULL AND service_keys IS NULL) OR
        (source = 'ADDITIONAL_SERVICE' AND service_keys IS NOT NULL AND array_length(service_keys, 1) > 0)
    )
);

-- Payment proof uploads
CREATE TABLE invoice_payment_proofs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id       UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    file_name        VARCHAR(255) NOT NULL,
    file_url         TEXT NOT NULL,
    file_size_bytes  INTEGER NOT NULL,
    bank_name        VARCHAR(100),
    reference_number VARCHAR(100),
    payment_date     DATE,
    uploaded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sts_sent_at      TIMESTAMPTZ
);

CREATE INDEX idx_invoices_customer_id     ON invoices(customer_id);
CREATE INDEX idx_invoices_nomination_id   ON invoices(nomination_id);
CREATE INDEX idx_invoices_status          ON invoices(status);
CREATE INDEX idx_invoices_source          ON invoices(source);
CREATE INDEX idx_invoices_due_date        ON invoices(due_date);
CREATE INDEX idx_invoices_parent_epb_id   ON invoices(parent_epb_id) WHERE parent_epb_id IS NOT NULL;
CREATE INDEX idx_invoice_proofs_invoice_id ON invoice_payment_proofs(invoice_id);
```

## 3. STS Platform Webhook — Inbound

### Endpoint: POST /api/webhooks/sts/invoice

**Auth:** HMAC-SHA256 signature in `X-STS-Signature` header (shared secret `STS_WEBHOOK_SECRET`).

**Events:**
- `EPB_SHORTFALL_DETECTED` — create row source=EPB_SHORTFALL
- `ADDITIONAL_SERVICE_INVOICE` — create row source=ADDITIONAL_SERVICE
- `INVOICE_PAYMENT_REJECT` — update existing row
- `INVOICE_PAID` — update existing row

**Payload examples:**

```json
// EPB_SHORTFALL_DETECTED
{
  "lps_nomination_id": "uuid",
  "event": "EPB_SHORTFALL_DETECTED",
  "timestamp": "ISO-8601",
  "data": {
    "invoice_number": "INV-20260505-00001",
    "parent_epb_number": "EPB-2026-0042",
    "amount": 5000000.00,
    "currency": "IDR",
    "due_date": "2026-05-20T23:59:59Z",
    "idempotency_key": "shortfall-EPB-2026-0042"
  }
}

// ADDITIONAL_SERVICE_INVOICE
{
  "lps_nomination_id": "uuid",
  "event": "ADDITIONAL_SERVICE_INVOICE",
  "timestamp": "ISO-8601",
  "data": {
    "invoice_number": "INV-20260505-00002",
    "service_keys": ["tank_cleaning", "bunkering"],
    "amount": 12000000.00,
    "currency": "IDR",
    "due_date": "2026-05-25T23:59:59Z",
    "idempotency_key": "addsvc-NOM-20260502-0001-tankcleaning-bunkering"
  }
}

// INVOICE_PAYMENT_REJECT
{
  "lps_invoice_id": "uuid",
  "event": "INVOICE_PAYMENT_REJECT",
  "timestamp": "ISO-8601",
  "data": {
    "rejection_reason": "Bukti transfer tidak terbaca."
  }
}

// INVOICE_PAID
{
  "lps_invoice_id": "uuid",
  "event": "INVOICE_PAID",
  "timestamp": "ISO-8601",
  "data": {
    "confirmed_at": "ISO-8601"
  }
}
```

**Processing per event:**

`EPB_SHORTFALL_DETECTED` / `ADDITIONAL_SERVICE_INVOICE` (Create):
1. Idempotency check: cari row existing dengan `sts_idempotency_key = idempotency_key`. Jika ada → return 200 (no-op).
2. Resolve `customer_id` dari `lps_nomination_id`.
3. Untuk source EPB_SHORTFALL: resolve `parent_epb_id` dari `parent_epb_number`.
4. Insert row ke `invoices` dengan status `UNPAID`.
5. Send notification ke customer: "Tagihan Invoice baru: {amount}. {Source-specific message}."

`INVOICE_PAYMENT_REJECT`:
1. Find `invoices` by `id = lps_invoice_id`.
2. Update `status = PAYMENT_REJECT`, store `rejection_reason`.
3. Send notification dengan rejection reason.

`INVOICE_PAID`:
1. Find `invoices` by `id = lps_invoice_id`.
2. Update `status = PAID`, set `confirmed_at`.
3. Send notification: "Pembayaran Invoice {invoice_number} dikonfirmasi."

Return 200 OK immediately.

## 4. Upload Payment Proof

### Endpoint: POST /api/customer/invoices/:invoice_id/proof

**Auth:** Customer JWT. Guard: `invoice` must belong to authenticated customer.

**Allowed statuses:** `UNPAID` (Bayar flow) atau `PAYMENT_REJECT` (Revisi Data flow).

Multipart form fields:
- `file` — required, PDF/JPG/PNG, max 5MB
- `bank_name` — optional string
- `reference_number` — optional string
- `payment_date` — optional date (YYYY-MM-DD)

> Tidak ada field `paid_amount` — Invoice harus dibayar penuh sesuai `invoices.amount`.

Logic:
1. Validate file type and size.
2. Save file to `uploads/invoices/:invoice_id/`.
3. Insert row into `invoice_payment_proofs`.
4. Update `invoices.status = WAITING_PAYMENT_VERIFICATION`, `rejection_reason = NULL`, `updated_at = now()`.
5. Async: send proof to STS Platform (Step 5).
6. Return 201.

## 5. Send Payment Proof to STS Platform

```
POST {STS_PLATFORM_BASE_URL}/api/invoices/{invoice_number}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_invoice_id": "uuid",
  "invoice_number": "INV-20260505-00001",
  "amount": 5000000.00,
  "currency": "IDR",
  "file_url": "https://...",
  "file_name": "bukti_bayar_invoice.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-789012",
  "payment_date": "2026-05-10",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff.

## 6. List & Detail Invoice

### Endpoint: GET /api/customer/invoices

**Auth:** Customer JWT. Returns all invoices belonging to the customer.

**Query params:**
- `status` (optional): filter by status atau kategori `ACTION_REQUIRED`
- `source` (optional): `EPB_SHORTFALL` | `ADDITIONAL_SERVICE`

Response:
```json
[
  {
    "id": "uuid",
    "invoice_number": "INV-20260505-00001",
    "nomination_id": "uuid",
    "nomination_number": "NOM-20260502-0001",
    "source": "EPB_SHORTFALL",
    "parent_epb_number": "EPB-2026-0042",     // jika source=EPB_SHORTFALL
    "service_keys": null,                       // jika source=ADDITIONAL_SERVICE: array
    "amount": 5000000.00,
    "currency": "IDR",
    "due_date": "2026-05-20T23:59:59Z",
    "is_overdue": false,
    "status": "UNPAID",
    "rejection_reason": null,
    "confirmed_at": null,
    "latest_proof": null,
    "updated_at": "2026-05-05T10:00:00Z"
  }
]
```

### Endpoint: GET /api/customer/invoices/:id

Full detail termasuk semua proof attempts, source-specific info (parent EPB detail jika shortfall, atau service labels jika additional service).

### Endpoint: GET /api/customer/invoices/summary

```json
{
  "action_required": 2,
  "waiting_verification": 1,
  "paid": 3
}
```

## 7. API Endpoints Summary

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/invoices | List all invoices for customer | Customer JWT |
| GET | /api/customer/invoices/summary | Aggregated counts per kategori | Customer JWT |
| GET | /api/customer/invoices/:id | Invoice detail + proof history | Customer JWT |
| POST | /api/customer/invoices/:id/proof | Upload payment proof | Customer JWT |
| POST | /api/webhooks/sts/invoice | Receive STS Invoice webhook (4 events) | HMAC |

## 8. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/billing | BillingLandingPage | Redirect ke /customer/billing/epb (default tab) |
| /customer/billing/invoice | InvoiceListPage | Tab "Invoice" — list semua Invoice customer dengan KPI pills + tabs filter |
| /customer/billing/invoice/:id | InvoiceDetailPage | Detail view + Bayar / Revisi Data actions, link ke parent (EPB atau nominasi) |

## 9. Service Key Mapping (untuk Display)

| Service Key | Label ID (UI) |
|---|---|
| `tank_cleaning` | Tank Cleaning |
| `bunkering` | Pengisian Bahan Bakar atau Air Bersih |
| `short_stay_temporary` | Short Stay Temporary |
| `supply_logistic` | Supply Logistic |
| `lay_up` | Lay Up |
| `ship_chandler` | Ship Chandler |
| `kapal_emergency` | Kapal Emergency |
