# Replit Handoff — M9: Nomination Status & Payment

## Context
Continuation of the LPS Customer Portal. **Module 9** handles everything after a nomination is submitted: receiving status updates from STS Platform, displaying EPB and schedule, customer revision flow, payment proof upload, and receiving payment verification results. Requires M7 (auth) and M8 (nominations) to be complete.

## Prerequisites
- M7 complete: JWT auth, `customers` table.
- M8 complete: `nominations` and `nomination_documents` tables, nomination CRUD API.
- STS Platform webhook URL must be registered with STS team: `POST https://<lps-domain>/api/webhooks/sts/nomination-status`
- Env vars needed: `STS_WEBHOOK_SECRET` (for HMAC signature verification), `STS_API_KEY`, `STS_PLATFORM_BASE_URL`

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_nomination_status_payment.up.sql`

```sql
-- Extend nominations table
ALTER TABLE nominations
    ADD COLUMN anchor_point             VARCHAR(50),
    ADD COLUMN etb                      TIMESTAMPTZ,
    ADD COLUMN estimated_duration_hours INTEGER,
    ADD COLUMN revision_notes           TEXT;

-- EPB data from STS Platform
CREATE TABLE nomination_epb (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_number    VARCHAR(50) NOT NULL,
    amount        DECIMAL(20,2) NOT NULL,
    currency      VARCHAR(10) NOT NULL DEFAULT 'IDR',
    due_date      TIMESTAMPTZ,
    received_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Payment proofs uploaded by customer
CREATE TABLE nomination_payment_proofs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id    UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    file_name        VARCHAR(255) NOT NULL,
    file_url         TEXT NOT NULL,
    file_size_bytes  INTEGER NOT NULL,
    uploaded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sts_sent_at      TIMESTAMPTZ,
    rejection_reason TEXT
);

-- Immutable status change history
CREATE TABLE nomination_status_history (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id UUID NOT NULL REFERENCES nominations(id),
    from_status   VARCHAR(50),
    to_status     VARCHAR(50) NOT NULL,
    triggered_by  VARCHAR(50) NOT NULL,  -- 'customer' | 'sts_webhook' | 'system'
    notes         TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_epb_nomination_id          ON nomination_epb(nomination_id);
CREATE INDEX idx_payment_proofs_nom_id      ON nomination_payment_proofs(nomination_id);
CREATE INDEX idx_status_history_nom_id      ON nomination_status_history(nomination_id);
```

Extended nomination statuses (add to existing): `APPROVED`, `NEED_REVISION`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_CONFIRMED`, `PAYMENT_REJECTED`

---

## Step 2: Backend — Status History Helper

File: `internal/nomination/status.go`

```go
func RecordStatusChange(ctx context.Context, db *gorm.DB, nominationID uuid.UUID, from, to, triggeredBy, notes string) error {
    history := NominationStatusHistory{
        ID:           uuid.New(),
        NominationID: nominationID,
        FromStatus:   &from,
        ToStatus:     to,
        TriggeredBy:  triggeredBy,
        Notes:        &notes,
        CreatedAt:    time.Now(),
    }
    return db.WithContext(ctx).Create(&history).Error
}
```

Call this function every time nomination status changes.

---

## Step 3: Backend — STS Webhook Handler

File: `internal/webhook/sts_handler.go`

### POST /api/webhooks/sts/nomination-status

**Step 3a: Verify HMAC signature**
```go
func verifySTSSignature(body []byte, signature, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}
```
Read raw body. Check `X-STS-Signature` header. Return 401 if invalid.

**Step 3b: Parse payload and route by event**

```go
type STSWebhookPayload struct {
    LPSNominationID  string          `json:"lps_nomination_id"`
    STSNominationID  string          `json:"sts_nomination_id"`
    Event            string          `json:"event"`
    Timestamp        time.Time       `json:"timestamp"`
    Data             json.RawMessage `json:"data"`
}
```

Route by `Event`:

**Event: `APPROVED`**
```go
type ApprovedData struct {
    AnchorPoint             string    `json:"anchor_point"`
    ETB                     time.Time `json:"etb"`
    EstimatedDurationHours  int       `json:"estimated_duration_hours"`
    EPB struct {
        EPBNumber string    `json:"epb_number"`
        Amount    float64   `json:"amount"`
        Currency  string    `json:"currency"`
        DueDate   time.Time `json:"due_date"`
    } `json:"epb"`
}
```
Actions:
1. Update nomination: `status = APPROVED`, `anchor_point`, `etb`, `estimated_duration_hours`, `sts_nomination_id`.
2. Insert into `nomination_epb`.
3. Record status history.
4. Send notification to customer (in-app + email).

**Event: `NEED_REVISION`**
```go
type NeedRevisionData struct {
    RevisionNotes string `json:"revision_notes"`
}
```
Actions:
1. Update nomination: `status = NEED_REVISION`, `revision_notes`.
2. Record status history.
3. Send notification.

**Event: `PAYMENT_CONFIRMED`**
Actions:
1. Update nomination: `status = PAYMENT_CONFIRMED`.
2. Record status history.
3. Send notification ("Pembayaran dikonfirmasi. Voyage dapat dimulai.").

**Event: `PAYMENT_REJECTED`**
```go
type PaymentRejectedData struct {
    RejectionReason string `json:"rejection_reason"`
}
```
Actions:
1. Update nomination: `status = PAYMENT_REJECTED`.
2. Update latest `nomination_payment_proofs` row: set `rejection_reason`.
3. Record status history.
4. Send notification with rejection reason.

Return 200 OK after all processing.

---

## Step 4: Backend — Nomination Status API

File: `internal/nomination/status_handler.go`

### GET /api/customer/nominations/:id/status
Returns full nomination status data including EPB (if APPROVED+), schedule, latest payment proof, and status history.

Response:
```json
{
  "nomination": { ...all fields... },
  "epb": { "epb_number": "", "amount": 0, "currency": "IDR", "due_date": "" } | null,
  "payment_proof": { "file_name": "", "uploaded_at": "", "rejection_reason": "" } | null,
  "status_history": [ { "from_status": "", "to_status": "", "triggered_by": "", "created_at": "" } ]
}
```

Guard: nomination must belong to authenticated customer.

---

## Step 5: Backend — Revision Submit

### POST /api/customer/nominations/:id/revise
Only allowed if `status = NEED_REVISION`.

Request body: same as nomination update (all nomination fields + documents can be re-uploaded).

Logic:
1. Validate all fields (same as submit).
2. Update nomination fields.
3. Set `status = SUBMITTED`, `submitted_at = now()`.
4. Async: re-send to STS Platform with updated payload (same endpoint as M8 Step 5 but include `sts_nomination_id` as a reference).
5. On STS success: set `status = PENDING`.
6. Record status history.

---

## Step 6: Backend — Payment Proof Upload

### POST /api/customer/nominations/:id/payment-proof
Only allowed if `status = APPROVED`.

Multipart form: `file` field.

Validation:
- File type: PDF, JPG, PNG only (check MIME type with `http.DetectContentType`)
- Max size: 5MB (5 × 1024 × 1024 bytes)
- Nomination must belong to authenticated customer
- Nomination status must be `APPROVED`

Logic:
1. Save file to storage.
2. Insert into `nomination_payment_proofs`.
3. Update `nominations.status = WAITING_PAYMENT_VERIFICATION`.
4. Record status history (triggered_by: "customer").
5. Async: send proof to STS Platform (Step 7).
6. Return 201 with file metadata.

---

## Step 7: Backend — Send Payment Proof to STS

File: `internal/sts/payment_client.go`

```
POST {STS_PLATFORM_BASE_URL}/api/nominations/{sts_nomination_id}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "file_url": "https://...",
  "file_name": "bukti_bayar.pdf",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff. On all retries fail: log error; status stays `WAITING_PAYMENT_VERIFICATION` (customer is NOT notified of the internal send failure — do not confuse with payment rejection).

---

## Step 8: Frontend — Nomination Status Page

File: `src/pages/customer/NominationStatusPage.tsx`
Route: `/customer/nominations/:id`

Use TanStack React Query to fetch `/api/customer/nominations/:id/status`. Poll every 30 seconds while status is `PENDING` or `WAITING_PAYMENT_VERIFICATION`.

**Status Banner (top of page, color-coded):**
| Status | Color | Display Text |
|--------|-------|-------------|
| PENDING | Blue | "Menunggu proses di STS Platform" |
| APPROVED | Green | "Nominasi Disetujui" |
| NEED_REVISION | Orange | "Perlu Revisi — [revision_notes]" |
| WAITING_PAYMENT_VERIFICATION | Purple | "Menunggu Verifikasi Pembayaran" |
| PAYMENT_CONFIRMED | Dark Green | "Pembayaran Dikonfirmasi — Voyage dapat dimulai" |
| PAYMENT_REJECTED | Red | "Pembayaran Ditolak — [rejection_reason]" |
| SUBMIT_FAILED | Red | "Gagal Mengirim ke STS Platform" with Retry button |

**Sections (show/hide based on status):**

*Nomination Details* — always visible: vessel name, ETA, cargo type, qty, charterer, barge count, documents list.

*Schedule & EPB* — visible only when status is APPROVED or later:
- Anchor Point, ETB (formatted), Estimated Duration
- EPB Number, Amount (format as "Rp XX.XXX.XXX"), Due Date

*Payment Proof Upload* — visible only when status is APPROVED:
- File dropzone accepting PDF/JPG/PNG, max 5MB
- Shows "Upload Bukti Pembayaran" button
- After upload: shows uploaded file name + timestamp

*Payment Proof Re-upload* — visible only when status is PAYMENT_REJECTED:
- Show rejection reason
- File dropzone for re-upload

*Revision Form Link* — visible only when status is NEED_REVISION:
- "Revisi Nominasi" button → navigate to `/customer/nominations/:id/revise`

---

## Step 9: Frontend — Revision Form

File: `src/pages/customer/NominationRevisionPage.tsx`
Route: `/customer/nominations/:id/revise`

Pre-fill form with existing nomination data. Same form as M8 NominationFormPage but in "revision" mode:
- Show revision notes from STS at the top as a warning banner
- All fields editable
- Submit button: "Re-Submit Nominasi"
- `POST /api/customer/nominations/:id/revise` on submit

---

## Acceptance Checklist
- [ ] STS webhook endpoint validates HMAC signature (rejects invalid signatures with 401)
- [ ] APPROVED webhook: nomination status updates to APPROVED, EPB and schedule stored and displayed
- [ ] NEED_REVISION webhook: status updates, revision notes displayed, revision form accessible
- [ ] PAYMENT_CONFIRMED webhook: status updates to PAYMENT_CONFIRMED
- [ ] PAYMENT_REJECTED webhook: status updates, rejection reason displayed, re-upload available
- [ ] Status changes within 1 minute of STS webhook receipt
- [ ] Customer receives in-app notification on every status change
- [ ] EPB section only visible when status is APPROVED or later
- [ ] Payment proof upload accepts PDF/JPG/PNG, rejects other types
- [ ] File > 5MB is rejected with clear error
- [ ] After payment proof upload, status immediately changes to WAITING_PAYMENT_VERIFICATION
- [ ] Payment proof is sent to STS Platform API
- [ ] Revision form is only accessible when status is NEED_REVISION
- [ ] Re-submission sets status back to PENDING
- [ ] All status changes are recorded in nomination_status_history
