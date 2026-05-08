# M9b — EPB & Invoice (Customer Portal): Technical Specifications

## 1. Payment Status Lifecycle

```
[M9 EPB Confirmation Submit]
        │
        ▼
    UNPAID ──► Click Pay ──► Upload + Submit
        │                           │
        │                           ▼
        │                    PENDING_REVIEW ──► STS verifies
        │                           │
        │               ┌───────────┴────────────┐
        │               ▼                        ▼
        │        PAYMENT_REJECT              PAID
        │         (click Revision Data)    (view-only)
        │               │
        │               ▼
        │       Upload + Re-Submit
        │               │
        └───────────────┘ (loop back to PENDING_REVIEW)
```

Statuses: `UNPAID`, `PENDING_REVIEW`, `PAYMENT_REJECT`, `PAID`

## 2. Database Schema

```sql
-- Core payment table (owned by M9b; first row inserted by M9 Step 6)
CREATE TABLE epb_payments (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id     UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_id            UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    status            VARCHAR(30) NOT NULL DEFAULT 'UNPAID',
    -- 'UNPAID' | 'PENDING_REVIEW' | 'PAYMENT_REJECT' | 'PAID'
    rejection_reason  TEXT,
    confirmed_at      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Payment proof uploads (one per attempt; multiple rows if rejected & re-uploaded)
CREATE TABLE epb_payment_proofs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epb_payment_id   UUID NOT NULL REFERENCES epb_payments(id) ON DELETE CASCADE,
    file_name        VARCHAR(255) NOT NULL,
    file_url         TEXT NOT NULL,
    file_size_bytes  INTEGER NOT NULL,
    bank_name        VARCHAR(100),
    reference_number VARCHAR(100),
    payment_date     DATE,
    uploaded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sts_sent_at      TIMESTAMPTZ
);

CREATE INDEX idx_epb_payments_nomination_id ON epb_payments(nomination_id);
CREATE INDEX idx_epb_payments_status        ON epb_payments(status);
CREATE INDEX idx_epb_proofs_payment_id      ON epb_payment_proofs(epb_payment_id);
```

## 3. STS Platform Webhook — Inbound (FR-EI-07)

### Endpoint: POST /api/webhooks/sts/epb-payment-status

**Auth:** HMAC-SHA256 signature in `X-STS-Signature` header (shared secret, same `STS_WEBHOOK_SECRET`).

**Payload:**
```json
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "event": "EPB_PENDING_REVIEW | EPB_PAYMENT_REJECT | EPB_PAID",
  "timestamp": "ISO-8601",
  "data": {
    // For EPB_PAYMENT_REJECT:
    "rejection_reason": "Jumlah pembayaran tidak sesuai EPB.",
    // For EPB_PAID:
    "confirmed_at": "ISO-8601"
  }
}
```

**Processing per event:**

`EPB_PENDING_REVIEW`:
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Update `status = PENDING_REVIEW`.
3. Send in-app + email notification to customer.

`EPB_PAYMENT_REJECT`:
1. Update `epb_payments.status = PAYMENT_REJECT`, store `rejection_reason`.
2. Send notification with rejection reason.

`EPB_PAID`:
1. Update `epb_payments.status = PAID`, set `confirmed_at`.
2. Send notification: "Pembayaran dikonfirmasi. Voyage dapat dimulai."

Return 200 OK immediately.

## 4. Upload Payment Proof (FR-EI-03, FR-EI-05, FR-EI-08)

### Endpoint: POST /api/customer/epb-payments/:epb_payment_id/proof

**Auth:** Customer JWT. Guard: `epb_payment` must belong to authenticated customer.

**Allowed statuses:** `UNPAID` (initial Pay flow) or `PAYMENT_REJECT` (Revision Data flow).

Multipart form fields:
- `file` — required, PDF/JPG/PNG, max 5MB
- `bank_name` — optional string
- `reference_number` — optional string
- `payment_date` — optional date (YYYY-MM-DD)

Logic:
1. Validate file type (MIME check) and size.
2. Save file to `uploads/epb-payments/:epb_payment_id/`.
3. Insert row into `epb_payment_proofs`.
4. Update `epb_payments.status = PENDING_REVIEW`, `updated_at = now()`.
5. Async: send to STS Platform (Step 5).
6. Return 201.

## 5. Send Payment Proof to STS Platform (FR-EI-08)

```
POST {STS_PLATFORM_BASE_URL}/api/epb/{epb_number}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "file_url": "https://...",
  "file_name": "bukti_bayar.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-123456",
  "payment_date": "2026-05-01",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff. On all retries fail: log error; `status` stays `PENDING_REVIEW` (internal STS send failure is NOT shown to customer as payment rejection).

## 6. List EPB & Invoice

### Endpoint: GET /api/customer/epb-payments

**Auth:** Customer JWT. Returns all EPB payments belonging to the customer.

Response:
```json
[
  {
    "id": "uuid",
    "nomination_id": "uuid",
    "nomination_number": "NOM-20260502-0001",
    "epb_number": "EPB-2026-0042",
    "amount": 15000000.00,
    "currency": "IDR",
    "due_date": "2026-04-30T23:59:59Z",
    "status": "PENDING_REVIEW",
    "rejection_reason": null,
    "confirmed_at": null,
    "latest_proof": {
      "file_name": "bukti_bayar.pdf",
      "uploaded_at": "2026-05-01T08:00:00Z"
    } | null,
    "updated_at": "2026-05-01T08:01:00Z"
  }
]
```

### Endpoint: GET /api/customer/epb-payments/:id

Returns full detail of a single EPB payment including all proof upload attempts.

## 7. API Endpoints Summary

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/epb-payments | List all EPBs for customer | Customer JWT |
| GET | /api/customer/epb-payments/:id | EPB detail + proof history | Customer JWT |
| POST | /api/customer/epb-payments/:id/proof | Upload payment proof (Pay or Revision Data) | Customer JWT |
| POST | /api/webhooks/sts/epb-payment-status | Receive STS payment status webhook | HMAC |

## 8. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/epb-invoice | EPBInvoiceListPage | List of all EPBs with status badges |
| /customer/epb-invoice/:id | EPBInvoiceDetailPage | Detail view + Pay / Revision Data actions |
