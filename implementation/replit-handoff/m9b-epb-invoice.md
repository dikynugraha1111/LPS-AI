# Replit Handoff — M9b: EPB & Invoice (Customer Portal)

**Version:** 1.0

## Context

Continuation of the LPS Customer Portal. **Module 9b** is the EPB & Invoice menu — it manages the full payment verification cycle after a customer's first proof of payment has been submitted via M9 (EPB Confirmation). Requires M7 (auth), M8 (nominations), and M9 (EPB Confirmation) to be complete.

The data flow is:
1. M9 webhook handler (event APPROVED) → otomatis buat `epb_payments` row (status `UNPAID`) tanpa aksi customer.
2. Customer buka M9b, lihat EPB berstatus UNPAID, klik "Pay" → upload proof → status berubah ke `WAITING_PAYMENT_VERIFICATION` (di `epb_payments` DAN `nominations`).
2. M9b (this module): customer pays → STS verifies → loop until `PAID`.

## Prerequisites
- M7 complete: JWT auth, `customers` table.
- M8 complete: `nominations`, `nomination_documents` tables.
- M9 complete: `nomination_epb`, `nomination_status_history` tables; STS webhook APPROVED handler berfungsi dan otomatis membuat row `epb_payments` (status `UNPAID`) untuk setiap nominasi yang disetujui.
- Env vars (same as M9): `STS_WEBHOOK_SECRET`, `STS_API_KEY`, `STS_PLATFORM_BASE_URL`

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_epb_payments_m9b.up.sql`

```sql
-- Core payment tracking table
-- NOTE: First row inserted by M9 webhook handler (event APPROVED) with status UNPAID
CREATE TABLE epb_payments (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id    UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_id           UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    status           VARCHAR(40) NOT NULL DEFAULT 'UNPAID',
    -- Allowed values: 'UNPAID' | 'WAITING_PAYMENT_VERIFICATION' | 'PAYMENT_REJECT' | 'PAID'
    rejection_reason TEXT,
    confirmed_at     TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- One row per payment proof upload attempt (Unpaid → pay, or Payment Reject → revision)
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

---

## Step 2: Go Models

File: `internal/epbpayment/model.go`

```go
type EPBPayment struct {
    ID              uuid.UUID  `gorm:"type:uuid;primaryKey"`
    NominationID    uuid.UUID  `gorm:"type:uuid;not null"`
    EPBID           uuid.UUID  `gorm:"type:uuid;not null"`
    Status          string     `gorm:"not null;default:UNPAID"`
    RejectionReason *string
    ConfirmedAt     *time.Time
    CreatedAt       time.Time
    UpdatedAt       time.Time

    // Associations
    EPB    NominationEPB      `gorm:"foreignKey:EPBID"`
    Proofs []EPBPaymentProof  `gorm:"foreignKey:EPBPaymentID"`
}

type EPBPaymentProof struct {
    ID              uuid.UUID `gorm:"type:uuid;primaryKey"`
    EPBPaymentID    uuid.UUID `gorm:"type:uuid;not null"`
    FileName        string    `gorm:"not null"`
    FileURL         string    `gorm:"not null"`
    FileSizeBytes   int       `gorm:"not null"`
    BankName        *string
    ReferenceNumber *string
    PaymentDate     *time.Time
    UploadedAt      time.Time `gorm:"default:now()"`
    STSSentAt       *time.Time
}
```

---

## Step 3: Backend — STS Webhook Handler (EPB payment events)

File: `internal/webhook/sts_epb_handler.go`

### POST /api/webhooks/sts/epb-payment-status

Use the same HMAC verification helper from M9 (`verifySTSSignature`).

**Payload:**
```go
type STSEPBWebhookPayload struct {
    LPSNominationID string          `json:"lps_nomination_id"`
    EPBNumber       string          `json:"epb_number"`
    Event           string          `json:"event"`
    Timestamp       time.Time       `json:"timestamp"`
    Data            json.RawMessage `json:"data"`
}
```

Route by `Event`:

> `EPB_PENDING_REVIEW` is NOT handled here — M9 sets status directly to `WAITING_PAYMENT_VERIFICATION` when proof is uploaded.

**Event: `EPB_PAYMENT_REJECT`**
```go
type EPBPaymentRejectData struct {
    RejectionReason string `json:"rejection_reason"`
}
```
```
Actions:
1. Find epb_payments.
2. Update status = 'PAYMENT_REJECT', rejection_reason = data.rejection_reason.
3. Send notification: "Pembayaran ditolak: [rejection_reason]. Silakan upload ulang."
4. Return 200 OK.
```

**Event: `EPB_PAID`**
```go
type EPBPaidData struct {
    ConfirmedAt time.Time `json:"confirmed_at"`
}
```
```
Actions:
1. Find epb_payments.
2. Update status = 'PAID', confirmed_at = data.confirmed_at.
3. Send notification: "Pembayaran dikonfirmasi. Voyage dapat dimulai."
4. Return 200 OK.
```

---

## Step 4: Backend — List EPB Payments

File: `internal/epbpayment/handler.go`

### GET /api/customer/epb-payments

Auth: Customer JWT. Returns all EPB payments for the authenticated customer.

Query:
```sql
SELECT ep.*, ne.epb_number, ne.amount, ne.currency, ne.due_date,
       n.nomination_number,
       (SELECT row_to_json(p) FROM epb_payment_proofs p
        WHERE p.epb_payment_id = ep.id ORDER BY p.uploaded_at DESC LIMIT 1) AS latest_proof
FROM epb_payments ep
JOIN nominations n ON ep.nomination_id = n.id
JOIN nomination_epb ne ON ep.epb_id = ne.id
WHERE n.customer_id = $1
ORDER BY ep.updated_at DESC
```

Response: array of EPB payment summaries (see specifications.md §6 for shape).

### GET /api/customer/epb-payments/:id

Returns full detail: EPB data + all proof upload attempts (ordered by uploaded_at DESC).

Guard: epb_payment must belong to authenticated customer (join check via nominations.customer_id).

---

## Step 5: Backend — Upload Payment Proof (Pay & Revision Data)

### POST /api/customer/epb-payments/:id/proof

Auth: Customer JWT.

Validation:
- `epb_payment.nomination.customer_id` must equal JWT customer ID.
- `epb_payment.status` must be `UNPAID` (Pay flow) atau `PAYMENT_REJECT` (Revision Data flow).
- File: PDF/JPG/PNG only, max 5MB.

Logic:
1. Validate file MIME type (`http.DetectContentType`).
2. Save file: `uploads/epb-payments/:epb_payment_id/:timestamp_filename`.
3. Insert `epb_payment_proofs` row.
4. Dalam **satu transaksi DB**:
   - Update `epb_payments.status = 'WAITING_PAYMENT_VERIFICATION'`, `rejection_reason = null`, `updated_at = now()`.
   - Update `nominations.status = 'WAITING_PAYMENT_VERIFICATION'` (join via `epb_payments.nomination_id`).
5. Async: send proof to STS Platform (Step 6).
6. Return 201 with proof metadata.

---

## Step 6: Backend — Send Proof to STS Platform

File: `internal/sts/epb_payment_client.go`

```
POST {STS_PLATFORM_BASE_URL}/api/epb/{epb_number}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body (JSON):
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "file_url": "https://storage.lps.internal/...",
  "file_name": "bukti_bayar.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-123456",
  "payment_date": "2026-05-01",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff (0.5s, 1s, 2s). On all retries fail: log error + set `epb_payment_proofs.sts_sent_at = null` (leave it null as indicator). Do NOT surface this failure to the customer — `status` stays `WAITING_PAYMENT_VERIFICATION`.

On success: update `epb_payment_proofs.sts_sent_at = now()`.

---

## Step 7: Frontend — EPB Invoice List Page

File: `src/pages/customer/EPBInvoiceListPage.tsx`
Route: `/customer/epb-invoice`

Use TanStack React Query to fetch `GET /api/customer/epb-payments`. Refetch every 30 seconds while any item has status `WAITING_PAYMENT_VERIFICATION`.

**Layout:**
- Page title: "EPB & Invoice"
- Cards list (one card per EPB):
  - Left: EPB Number, Nomination Number (small), Amount (bold IDR format), Due Date
  - Right: Status badge (color-coded), "Lihat Detail" link
  - If status `UNPAID`: show "Pay" button directly on card
  - If status `PAYMENT_REJECT`: show "Revision Data" button directly on card

**Status badge colors:**

| Status | Color | Label |
|--------|-------|-------|
| UNPAID | Red | Belum Dibayar |
| WAITING_PAYMENT_VERIFICATION | Yellow | Menunggu Verifikasi |
| PAYMENT_REJECT | Orange | Pembayaran Ditolak |
| PAID | Green | Lunas |

---

## Step 8: Frontend — EPB Invoice Detail Page

File: `src/pages/customer/EPBInvoiceDetailPage.tsx`
Route: `/customer/epb-invoice/:id`

Fetch `GET /api/customer/epb-payments/:id`. Poll every 30 seconds if status is `WAITING_PAYMENT_VERIFICATION`.

**Sections:**

*EPB Summary (always visible — view-only):*
- EPB Number, Amount (IDR), Due Date
- Related Nomination Number (link to nomination status page)
- Current status badge + updated_at timestamp

*Payment Action (only if UNPAID):*
- "Pay" button → opens upload modal

*Waiting Verification Info (only if WAITING_PAYMENT_VERIFICATION):*
- Info box: "Menunggu Verifikasi Pembayaran — bukti pembayaran Anda sedang diproses oleh tim Finance."
- No action buttons.

*Rejection Info (only if PAYMENT_REJECT):*
- Warning box: "Pembayaran Ditolak"
- Rejection reason text from STS
- "Revision Data" button → opens upload modal

*Confirmation Info (only if PAID):*
- Green success box: "Pembayaran telah dikonfirmasi"
- Confirmed At timestamp

*Proof Upload History (always visible after first proof):*
- Table: file name, upload timestamp, Bank Name, Reference Number, Payment Date
- Multiple rows if rejected and re-uploaded

**Upload Modal (shared for Pay dan Revision Data):**
- File dropzone: PDF/JPG/PNG, max 5MB
- Optional fields: Bank Name (text), Reference Number (text), Payment Date (date picker)
- Submit button: "Submit Pembayaran" (untuk UNPAID) / "Submit Revisi Pembayaran" (untuk PAYMENT_REJECT)
- Loading state during upload
- On success: close modal, refetch detail page, show toast "Bukti pembayaran berhasil diupload. Menunggu verifikasi."

---

## Step 9: Update Navigation

Add "EPB & Invoice" to the customer portal sidebar/navigation. Placed after "Nominasi Saya" (M8/M9) in the nav order.

File: `src/components/CustomerSidebar.tsx` (or equivalent nav component)

```tsx
{ label: 'EPB & Invoice', href: '/customer/epb-invoice', icon: ReceiptText }
```

Add route to router:
```tsx
<Route path="/customer/epb-invoice" component={EPBInvoiceListPage} />
<Route path="/customer/epb-invoice/:id" component={EPBInvoiceDetailPage} />
```

---

## Acceptance Checklist

**Database:**
- [ ] `epb_payments` table created with correct columns and FK constraints
- [ ] `epb_payment_proofs` table created with correct columns and FK constraints
- [ ] Indexes created on nomination_id, status, and epb_payment_id

**Webhook:**
- [ ] `POST /api/webhooks/sts/epb-payment-status` validates HMAC signature
- [ ] `EPB_PAYMENT_REJECT` event: status updates to `PAYMENT_REJECT`, rejection_reason stored
- [ ] `EPB_PAID` event: status updates to `PAID`, confirmed_at stored
- [ ] Customer receives in-app + email notification on every webhook event
- [ ] Status changes reflected in UI within 1 minute of webhook receipt
- [ ] `EPB_PENDING_REVIEW` event is NOT implemented (not needed — M9 sets status directly)

**API:**
- [ ] `GET /api/customer/epb-payments` returns only the authenticated customer's EPBs
- [ ] `GET /api/customer/epb-payments/:id` returns full detail + proof history
- [ ] `POST /api/customer/epb-payments/:id/proof` rejects file types other than PDF/JPG/PNG
- [ ] `POST /api/customer/epb-payments/:id/proof` rejects files > 5MB
- [ ] Upload allowed when status is `UNPAID` (Pay flow) atau `PAYMENT_REJECT` (Revision Data flow)
- [ ] After upload: `epb_payments.status` DAN `nominations.status` sama-sama berubah ke `WAITING_PAYMENT_VERIFICATION` dalam satu transaksi
- [ ] `rejection_reason` di-clear saat re-upload dari PAYMENT_REJECT
- [ ] Proof sent to STS Platform with retry (max 3×)
- [ ] Customer cannot access another customer's EPB payments

**Frontend:**
- [ ] EPB & Invoice list page shows all EPBs with correct status badges (UNPAID = red, WAITING_PAYMENT_VERIFICATION = yellow, PAYMENT_REJECT = orange, PAID = green)
- [ ] "Pay" button visible only for UNPAID status
- [ ] "Revision Data" button visible only for PAYMENT_REJECT status
- [ ] Detail page shows Pay action section for UNPAID status
- [ ] Detail page shows waiting info box for WAITING_PAYMENT_VERIFICATION status
- [ ] Detail page shows rejection reason for PAYMENT_REJECT status
- [ ] Upload modal shared untuk Pay dan Revision Data, submit label berbeda
- [ ] After successful upload: status badge updates to "Menunggu Verifikasi"
- [ ] Paid EPB shows no action buttons and displays confirmed_at timestamp
- [ ] Navigation item added to customer sidebar
- [ ] Routes registered and accessible
