# M9 — Nomination Status & Payment: Technical Specifications

## 1. Full Nomination Status Lifecycle

```
DRAFT ──────────────────────────────────────────── (M8)
  └─► SUBMITTED ──► PENDING ──────────────────────── (M8 → M9)
                       ├─► APPROVED ──────────────── (STS webhook)
                       │       └─► WAITING_PAYMENT_VERIFICATION (after upload)
                       │               ├─► PAYMENT_CONFIRMED ── (STS webhook → voyage starts)
                       │               └─► PAYMENT_REJECTED ─── (STS webhook → re-upload)
                       └─► NEED_REVISION ──► (customer revises) ──► PENDING (re-submit)
```

## 2. STS Platform Webhook (Inbound) (FR-NP-01, FR-NP-09, FR-NP-11)

### Endpoint: POST /api/webhooks/sts/nomination-status
**Auth:** HMAC-SHA256 signature in `X-STS-Signature` header (shared secret).

**Payload:**
```json
{
  "lps_nomination_id": "uuid",
  "sts_nomination_id": "sts-uuid",
  "event": "APPROVED | NEED_REVISION | PAYMENT_CONFIRMED | PAYMENT_REJECTED",
  "timestamp": "ISO-8601",
  "data": {
    // For APPROVED:
    "anchor_point": "AP-3",
    "etb": "2026-05-02T06:00:00Z",
    "estimated_duration_hours": 36,
    "epb": {
      "epb_number": "EPB-2026-0042",
      "amount": 15000000.00,
      "currency": "IDR",
      "due_date": "2026-04-30T23:59:59Z"
    },
    // For NEED_REVISION:
    "revision_notes": "Dokumen Rencana Kerja tidak lengkap.",
    // For PAYMENT_REJECTED:
    "rejection_reason": "Jumlah pembayaran tidak sesuai EPB."
  }
}
```

**Processing:**
1. Verify HMAC signature → reject with 401 if invalid.
2. Find nomination by `lps_nomination_id` → update status and store relevant data.
3. Trigger in-app + email notification to customer (FR-NP-04).
4. Return 200 OK immediately (process async if needed).

## 3. View EPB & Schedule (FR-NP-02, FR-NP-03)
- EPB and schedule details are visible only when `status = APPROVED` or later.
- EPB data stored in `nomination_epb` table (see schema below).
- Display fields: EPB Number, Amount (formatted IDR), Due Date, Anchor Point, ETB, Estimated Duration.

## 4. Need Revision Flow (FR-NP-05)
- When `status = NEED_REVISION`, customer sees a banner with `revision_notes` from STS.
- Customer can edit: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Est. Barge, re-upload documents.
- On re-submit: same flow as M8 FR-NS-05 (send updated payload to STS API).
- Status transitions back to `PENDING` after re-submit.

## 5. Payment Proof Upload (FR-NP-06, FR-NP-07, FR-NP-08)

### Upload Constraints
- Accepted formats: PDF, JPG, PNG
- Max file size: 5MB per file
- One payment proof per EPB (can re-upload to replace)

### Upload Flow
1. Customer uploads file → store in LPS file storage.
2. Immediately: set `status = WAITING_PAYMENT_VERIFICATION`.
3. Send to STS Platform: `POST /sts-api/nominations/:sts_id/payment-proof` with file URL + metadata.
4. On STS failure: retry max 3× exponential backoff. Notify customer if all retries fail.

### STS Payment Proof Payload
```json
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "file_url": "https://storage.lps.internal/...",
  "file_name": "bukti_bayar.pdf",
  "uploaded_at": "ISO-8601"
}
```

## 6. Payment Verification Result (FR-NP-09, FR-NP-10, FR-NP-11)
- Received via STS webhook (event: `PAYMENT_CONFIRMED` or `PAYMENT_REJECTED`).
- On `PAYMENT_CONFIRMED`: status → `PAYMENT_CONFIRMED`; display "Pembayaran dikonfirmasi. Voyage dapat dimulai."
- On `PAYMENT_REJECTED`: status → `PAYMENT_REJECTED`; display rejection reason; customer can re-upload.

## 7. Database Schema Additions

```sql
-- Extends nominations table (columns added via migration):
ALTER TABLE nominations ADD COLUMN anchor_point VARCHAR(50);
ALTER TABLE nominations ADD COLUMN etb TIMESTAMPTZ;
ALTER TABLE nominations ADD COLUMN estimated_duration_hours INTEGER;
ALTER TABLE nominations ADD COLUMN revision_notes TEXT;
ALTER TABLE nominations ADD COLUMN sts_nomination_id VARCHAR(100);

CREATE TABLE nomination_epb (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id   UUID NOT NULL REFERENCES nominations(id),
    epb_number      VARCHAR(50) NOT NULL,
    amount          DECIMAL(20,2) NOT NULL,
    currency        VARCHAR(10) NOT NULL DEFAULT 'IDR',
    due_date        TIMESTAMPTZ,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nomination_payment_proofs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id   UUID NOT NULL REFERENCES nominations(id),
    file_name       VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    sts_sent_at     TIMESTAMPTZ,
    rejection_reason TEXT
);

CREATE TABLE nomination_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id   UUID NOT NULL REFERENCES nominations(id),
    from_status     VARCHAR(50),
    to_status       VARCHAR(50) NOT NULL,
    triggered_by    VARCHAR(50) NOT NULL, -- 'customer' | 'sts_webhook' | 'system'
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 8. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/nominations/:id/status | Get current status + EPB + schedule | Customer JWT |
| POST | /api/customer/nominations/:id/revise | Submit revision | Customer JWT |
| POST | /api/customer/nominations/:id/payment-proof | Upload proof of payment | Customer JWT |
| POST | /api/webhooks/sts/nomination-status | Receive STS status webhook | HMAC |

## 9. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/nominations/:id | NominationStatusPage | Status, EPB, payment upload |
| /customer/nominations/:id/revise | NominationFormPage (edit mode) | Revision form |
