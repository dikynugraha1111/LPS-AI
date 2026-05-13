# Replit Handoff — M9: Nomination Status & EPB Confirmation

**Version:** 2.0 (revised per BRD v3.1 / Swimlane V3)

## Context

Continuation of the LPS Customer Portal. **Module 9** handles everything after a nomination is submitted up to the moment the customer submits their first proof of payment (EPB Confirmation). After that Submit, data is handed off to **M9b (EPB & Invoice)** for the full payment verification cycle. Requires M7 (auth) and M8 (nominations) to be complete.

> **Scope boundary:** This handoff does NOT implement WAITING_PAYMENT_VERIFICATION, PAYMENT_CONFIRMED, or PAYMENT_REJECTED states — those belong to M9b.

## Prerequisites
- M7 complete: JWT auth, `customers` table.
- M8 complete: `nominations` and `nomination_documents` tables, nomination CRUD API.
- STS Platform webhook URL must be registered with STS team: `POST https://<lps-domain>/api/webhooks/sts/nomination-status`
- Env vars needed: `STS_WEBHOOK_SECRET` (for HMAC signature verification), `STS_API_KEY`, `STS_PLATFORM_BASE_URL`

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca dua file ini:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — foundation tokens, component library, surface preset.
2. **Per-modul UI design:** [`implementation/design/m9-nomination-status-payment-ui.md`](../design/m9-nomination-status-payment-ui.md) — detail page 2-column layout, status banner varian per status, timeline pattern, EPB detail section, action card kontextual.

**Surface:** A — Customer Portal (Bahasa Indonesia).

**UI rules ringkas:**
- Status banner: pakai pattern dari design system §3.2 dengan variant sesuai status (Info/Success/Warning/Pending verify/Confirmed).
- Status mapping label ID: lihat design system §2.1 "Status mapping LPS" (Menunggu Review, Disetujui, Perlu Revisi, Menunggu Verifikasi Pembayaran, Pembayaran Dikonfirmasi).
- Timeline custom: vertical dot bullet + connector line, dot pulse animate untuk current step.
- Action card kontextual: hanya tampil saat ada aksi yang bisa diambil customer.
- Color primer: navy `#0F2A4D`. Canvas `bg-slate-50`. Font Inter.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_nomination_status_m9.up.sql`

```sql
-- Extend nominations table
ALTER TABLE nominations
    ADD COLUMN anchor_point             VARCHAR(50),
    ADD COLUMN etb                      TIMESTAMPTZ,
    ADD COLUMN estimated_duration_hours INTEGER,
    ADD COLUMN revision_notes           TEXT,
    ADD COLUMN sts_nomination_id        VARCHAR(100);

-- EPB data received from STS Platform (read-only display)
CREATE TABLE nomination_epb (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_number    VARCHAR(50) NOT NULL UNIQUE,
    amount        DECIMAL(20,2) NOT NULL,
    currency      VARCHAR(10) NOT NULL DEFAULT 'IDR',
    due_date      TIMESTAMPTZ,
    received_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Immutable status change history (used by M9 and M9b)
CREATE TABLE nomination_status_history (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id UUID NOT NULL REFERENCES nominations(id),
    from_status   VARCHAR(50),
    to_status     VARCHAR(50) NOT NULL,
    triggered_by  VARCHAR(50) NOT NULL,  -- 'customer' | 'sts_webhook' | 'system'
    notes         TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_epb_nomination_id       ON nomination_epb(nomination_id);
CREATE INDEX idx_status_history_nom_id   ON nomination_status_history(nomination_id);
```

Nomination statuses managed in M9: `PENDING`, `APPROVED`, `NEED_REVISION`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_CONFIRMED`

> Setelah customer upload proof di M9b, M9b backend ikut mengupdate `nominations.status = WAITING_PAYMENT_VERIFICATION`. Tidak ada status `EPB_CONFIRMATION_SUBMITTED`. Row `epb_payments` dibuat oleh M9 webhook handler (event APPROVED) dengan status `UNPAID`.

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

## Step 3: Backend — STS Webhook Handler (APPROVED / NEED_REVISION only)

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
    AnchorPoint            string    `json:"anchor_point"`
    ETB                    time.Time `json:"etb"`
    EstimatedDurationHours int       `json:"estimated_duration_hours"`
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
3. **Otomatis buat row `epb_payments`** (defined in M9b schema) dengan status `UNPAID`, linked ke `nomination_epb.id` yang baru dibuat. Ini memunculkan EPB di menu M9b tanpa aksi customer.
   ```sql
   INSERT INTO epb_payments (id, nomination_id, epb_id, status)
   VALUES (gen_random_uuid(), :nomination_id, :epb_id, 'UNPAID');
   ```
4. Record status history (triggered_by: "sts_webhook").
5. Send in-app + email notification to customer.

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

Return 200 OK after all processing.

> **Events NOT handled here:** `PAYMENT_CONFIRMED`, `PAYMENT_REJECTED` — these are routed to the M9b webhook handler.

---

## Step 4: Backend — Nomination Status API

File: `internal/nomination/status_handler.go`

### GET /api/customer/nominations/:id/status

Returns nomination status data including EPB (if APPROVED) and status history.

Response:
```json
{
  "nomination": {
    "id": "uuid",
    "nomination_number": "NOM-20260502-0001",
    "vessel_name": "MV Sinar Laut",
    "status": "APPROVED",
    "anchor_point": "AP-3",
    "etb": "2026-05-02T06:00:00Z",
    "estimated_duration_hours": 36,
    "revision_notes": null,
    "submitted_at": "2026-05-01T10:00:00Z"
  },
  "epb": {
    "epb_number": "EPB-2026-0042",
    "amount": 15000000.00,
    "currency": "IDR",
    "due_date": "2026-04-30T23:59:59Z"
  } | null,
  "status_history": [
    { "from_status": "SUBMITTED", "to_status": "PENDING", "triggered_by": "system", "created_at": "" }
  ]
}
```

Guard: nomination must belong to authenticated customer.

---

## Step 5: Backend — Revision Submit

### POST /api/customer/nominations/:id/revise

Only allowed if `status = NEED_REVISION`.

Request body: same nomination fields + documents can be re-uploaded.

Logic:
1. Validate all fields (same rules as M8 submit).
2. Update nomination fields.
3. Set `status = SUBMITTED`, `submitted_at = now()`.
4. Async: re-send to STS Platform with updated payload (same M8 endpoint, include `sts_nomination_id` as reference).
5. On STS success: set `status = PENDING`.
6. Record status history (triggered_by: "customer").

---

## Step 6: Frontend — Tombol "Bayar EPB" (Link ke M9b)

Tidak ada endpoint backend baru di M9 untuk upload proof. M9 hanya menyediakan tombol navigasi ke M9b.

Logika tampilan di `NominationStatusPage.tsx` (berdasarkan `nominations.status`):
- Jika `status = APPROVED`: tampilkan tombol **"Bayar EPB"** → navigasi ke `/customer/epb-invoice`.
- Jika `status = WAITING_PAYMENT_VERIFICATION`: sembunyikan tombol "Bayar EPB", tampilkan info banner teal:
  `"Bukti pembayaran sedang diverifikasi. Lihat status di menu EPB & Invoice →"` (link ke `/customer/epb-invoice`).

> Upload proof dilakukan sepenuhnya di M9b melalui `POST /api/customer/epb-payments/:id/proof`. M9b yang bertanggung jawab mengupdate `nominations.status` saat proof berhasil diupload.

---

## Step 7: Frontend — Nomination Status Page

File: `src/pages/customer/NominationStatusPage.tsx`
Route: `/customer/nominations/:id`

Use TanStack React Query to fetch `/api/customer/nominations/:id/status`. Poll every 30 seconds while status is `PENDING`.

**Status Banner (top of page, color-coded):**

| Status | Color | Display Text |
|--------|-------|-------------|
| PENDING | Blue | "Menunggu proses di STS Platform" |
| APPROVED | Green | "Nominasi Disetujui — Silakan lakukan pembayaran EPB" |
| WAITING_PAYMENT_VERIFICATION | Teal | "Bukti pembayaran sedang diverifikasi — Lihat status di menu EPB & Invoice →" |
| NEED_REVISION | Orange | "Perlu Revisi — [revision_notes]" |
| SUBMIT_FAILED | Red | "Gagal Mengirim ke STS Platform" with Retry button |

**Sections (show/hide based on status):**

*Nomination Details* — always visible: vessel name, ETA, cargo type, qty, charterer, barge count, documents list.

*Schedule & EPB* — visible when status is `APPROVED` or `WAITING_PAYMENT_VERIFICATION`:
- Anchor Point, ETB (formatted date-time), Estimated Duration
- EPB Number, Amount (format as "Rp XX.XXX.XXX"), Due Date
- Tombol **"Bayar EPB"** → link ke `/customer/epb-invoice` (hanya tampil saat `APPROVED`)

*Info Banner EPB* — visible when status is `WAITING_PAYMENT_VERIFICATION`:
- Teal info box: "Bukti pembayaran sedang diverifikasi." + link ke EPB & Invoice

*Revision Form Link* — visible only when status is `NEED_REVISION`:
- "Revisi Nominasi" button → navigate to `/customer/nominations/:id/revise`

---

## Step 8: Frontend — Revision Form

File: `src/pages/customer/NominationRevisionPage.tsx`
Route: `/customer/nominations/:id/revise`

Pre-fill form with existing nomination data. Same form as M8 NominationFormPage but in revision mode:
- Show revision notes from STS at the top as a warning banner
- All fields editable
- Submit button: "Re-Submit Nominasi"
- `POST /api/customer/nominations/:id/revise` on submit
- On success: redirect to `/customer/nominations/:id`

---

## Acceptance Checklist

- [ ] STS webhook endpoint validates HMAC signature (rejects invalid signatures with 401)
- [ ] `APPROVED` webhook: nomination status = `APPROVED`, EPB and schedule stored and displayed
- [ ] `APPROVED` webhook: row `epb_payments` otomatis dibuat dengan status `UNPAID` (linked ke nomination_epb)
- [ ] `NEED_REVISION` webhook: status updates, revision notes displayed, revision form accessible
- [ ] Status changes within 1 minute of STS webhook receipt
- [ ] Customer receives in-app + email notification on every status change
- [ ] EPB section (Schedule & EPB + tombol "Bayar EPB") visible when status is `APPROVED`
- [ ] Tombol "Bayar EPB" mengarah ke `/customer/epb-invoice`
- [ ] Saat status `WAITING_PAYMENT_VERIFICATION`: tombol "Bayar EPB" disembunyikan, info banner teal tampil dengan link ke EPB & Invoice
- [ ] Tidak ada endpoint upload proof di M9 — upload dilakukan di M9b
- [ ] Revision form is only accessible when status is `NEED_REVISION`
- [ ] Re-submission sets status back to `PENDING`
- [ ] All nomination status changes recorded in `nomination_status_history`
