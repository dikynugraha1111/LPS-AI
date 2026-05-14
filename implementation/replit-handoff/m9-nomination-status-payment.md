# Replit Handoff — M9: Nomination Status & EPB Confirmation

**Version:** 2.2 (v3.4 — EPB invoice-style preview + extended schema)

> **v2.2 changes (2026-05-14):** Tabel `nomination_epb` diperluas dengan kolom invoice-style (vessel ops, kalkulasi subtotal/ppn, bank info, kode bayar, pdf_url) + tabel baru `nomination_epb_line_items`. Webhook APPROVED payload menerima field tambahan dari STS. M9 detail page menampilkan **preview Invoice Detail Card (compact)** + tombol "Download EPB PDF". Tambah FR-NP-09.

> **v2.1 changes (2026-05-13):** Status labels disinkronkan ke BRD v3.3 mapping (UNPAID→Belum Dibayar, Pembayaran Dikonfirmasi→Lunas, Menunggu Verifikasi Pembayaran→Menunggu Verifikasi). Route ke M9b berubah dari `/customer/epb-invoice` → `/customer/billing/epb`. Saat insert row `epb_payments` di event APPROVED, sekarang juga set `total_amount`, `currency`, `due_date` (kolom baru di v3.3 — lihat M9b handoff v2.0).

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
- Status mapping label ID (v3.3 sync): lihat design system §2.1 "Status mapping LPS" (Menunggu Review, Disetujui, Perlu Revisi, **Menunggu Verifikasi**, **Lunas**). Label `UNPAID` → "Belum Dibayar"; `Pembayaran Dikonfirmasi` → "Lunas"; `Menunggu Verifikasi Pembayaran` → "Menunggu Verifikasi".
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
-- v2.2: extended dengan field invoice-style (vessel ops, kalkulasi, bank info, pdf_url)
CREATE TABLE nomination_epb (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id       UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_number          VARCHAR(50) NOT NULL UNIQUE,
    -- Kalkulasi
    subtotal            NUMERIC(20,2),
    vat_rate            NUMERIC(5,4) DEFAULT 0.11,
    vat_amount          NUMERIC(20,2),
    total_amount        NUMERIC(20,2) NOT NULL,   -- = subtotal + vat_amount; fallback legacy 'amount'
    currency            VARCHAR(10) NOT NULL DEFAULT 'IDR',  -- 'IDR' | 'USD'
    due_date            TIMESTAMPTZ,
    -- Operasional voyage
    vessel_name         VARCHAR(255),
    crane               VARCHAR(50),
    sts_slot            VARCHAR(50),
    mooring_team        VARCHAR(100),
    eta                 TIMESTAMPTZ,
    surveyor            VARCHAR(255),
    anchor              VARCHAR(50),
    est_duration_hours  INTEGER,
    -- Bank instruction
    bank_name           VARCHAR(100),
    bank_account_no     VARCHAR(50),
    bank_account_holder VARCHAR(255),
    kode_bayar          VARCHAR(100),
    -- Dokumen
    epb_pdf_url         TEXT,
    -- Audit
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Line items per EPB (v2.2)
CREATE TABLE nomination_epb_line_items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epb_id      UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    label       VARCHAR(255) NOT NULL,
    volume      VARCHAR(50),
    rate        VARCHAR(50),
    amount      NUMERIC(20,2) NOT NULL,
    sort_order  INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_epb_line_items_epb_id ON nomination_epb_line_items(epb_id);

-- ⚠️ Migrasi delta untuk DB yang sudah ada nomination_epb (v2.1):
-- Pakai ALTER TABLE … ADD COLUMN IF NOT EXISTS untuk seluruh kolom baru di atas.
-- Kolom `amount` lama dipertahankan sebagai alias (atau di-rename ke total_amount).
-- Backfill: untuk row existing, total_amount := amount, vat_amount := NULL, subtotal := NULL.

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

**Event: `APPROVED` (v2.2 — extended)**
```go
type VesselOps struct {
    VesselName       string    `json:"vessel_name"`
    Crane            string    `json:"crane"`
    STSSlot          string    `json:"sts_slot"`
    MooringTeam      string    `json:"mooring_team"`
    ETA              time.Time `json:"eta"`
    Surveyor         string    `json:"surveyor"`
    Anchor           string    `json:"anchor"`
    EstDurationHours int       `json:"est_duration_hours"`
}

type LineItem struct {
    Label  string  `json:"label"`
    Volume *string `json:"volume"`
    Rate   *string `json:"rate"`
    Amount float64 `json:"amount"`
}

type BankInfo struct {
    BankName      string `json:"bank_name"`
    AccountNumber string `json:"account_number"`
    AccountHolder string `json:"account_holder"`
    KodeBayar     string `json:"kode_bayar"`
}

type ApprovedData struct {
    AnchorPoint            string    `json:"anchor_point"`
    ETB                    time.Time `json:"etb"`
    EstimatedDurationHours int       `json:"estimated_duration_hours"`
    EPB struct {
        EPBNumber   string     `json:"epb_number"`
        // Kalkulasi
        Subtotal    *float64   `json:"subtotal"`
        VatRate     *float64   `json:"vat_rate"`
        VatAmount   *float64   `json:"vat_amount"`
        TotalAmount float64    `json:"total_amount"` // wajib
        Amount      *float64   `json:"amount"`       // legacy fallback (alias of total_amount)
        Currency    string     `json:"currency"`     // 'IDR' | 'USD'
        DueDate     time.Time  `json:"due_date"`
        // Invoice-style (v2.2)
        VesselOps   *VesselOps `json:"vessel_ops"`
        LineItems   []LineItem `json:"line_items"`
        BankInfo    *BankInfo  `json:"bank_info"`
        EPBPDFURL   *string    `json:"epb_pdf_url"`
    } `json:"epb"`
}
```
Actions:
1. Update nomination: `status = APPROVED`, `anchor_point`, `etb`, `estimated_duration_hours`, `sts_nomination_id`.
2. Insert into `nomination_epb` dengan **semua field extended** (`subtotal`, `vat_*`, `vessel_*`, `bank_*`, `kode_bayar`, `epb_pdf_url`). Jika field opsional null di payload, simpan NULL di kolom. Resolve `total_amount`: prefer `total_amount`, fallback ke `amount` legacy.
3. Insert rows into `nomination_epb_line_items` (jika `line_items[]` ada).
4. **Otomatis buat row `epb_payments`** (defined in M9b schema) dengan status `UNPAID`, linked ke `nomination_epb.id`. Set `total_amount`, `currency`, `due_date` dari EPB.
   ```sql
   INSERT INTO epb_payments (id, nomination_id, epb_id, status, total_amount, currency, due_date)
   VALUES (gen_random_uuid(), :nomination_id, :epb_id, 'UNPAID', :total, :currency, :due);
   ```
5. Record status history (triggered_by: "sts_webhook").
6. Send in-app + email notification to customer.

> **Backwards compatibility:** Jika STS belum upgrade ke v3.4 payload (field `vessel_ops`/`line_items`/dsb tidak ada), insert tetap berhasil dengan kolom NULL. UI fallback minimal di M9b + M9 (lihat per-modul UI doc).

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
    "epb_number": "EPB-20260430-00008",
    "epb_payment_id": "uuid",                       // dari epb_payments — utk navigate ke M9b
    "payment_status": "UNPAID",                     // status payment terkini (mirror dari epb_payments)
    "subtotal": 125000.00,
    "vat_rate": 0.11,
    "vat_amount": 13750.00,
    "total_amount": 138750.00,
    "currency": "USD",
    "due_date": "2026-03-05T23:59:59Z",
    "vessel_ops": {
      "vessel_name": "MV Pacific Star", "crane": "Crane 2", "sts_slot": "STS-202603-001",
      "mooring_team": "Team Alpha", "eta": "2026-03-12T14:00:00Z", "surveyor": "PT Sucofindo",
      "anchor": "Anchor 3", "est_duration_hours": 24
    } | null,
    "line_items": [
      { "label": "STS Fee", "volume": "50,000 MT", "rate": "$2.50/ton", "amount": 125000.00 }
    ] | [],
    "epb_pdf_url_available": true                   // boolean; URL aslinya tidak diekspose, hanya flag
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
- Jika `status = APPROVED`: tampilkan tombol **"Bayar EPB"** → navigasi ke `/customer/billing/epb`.
- Jika `status = WAITING_PAYMENT_VERIFICATION`: sembunyikan tombol "Bayar EPB", tampilkan info banner teal:
  `"Bukti pembayaran sedang diverifikasi. Lihat status di menu EPB & Invoice →"` (link ke `/customer/billing/epb`).

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

*Schedule & EPB Detail (v2.2 — invoice-style preview)* — visible when status is `APPROVED` or `WAITING_PAYMENT_VERIFICATION`:

Render pakai **Invoice Detail Card (compact variant)** dari design system §3.2 v1.3. Compact = Block 1 (Vessel Ops Grid) + Block 2 (Line Items Table dengan Subtotal/PPn/Total). **Block 3 Payment Instruction NOT shown** — instruksi pembayaran lengkap dilihat di M9b.

Card header:
- Kiri: `EPB-{epb_number}` (mono bold) + `<StatusBadge>` untuk `payment_status`.
- Kanan: tombol `<Button variant="outline"><Download /> Download EPB PDF</Button>` (FR-NP-09). Disabled jika `epb_pdf_url_available = false`. Klik → `window.open('/api/customer/epb-payments/'+epb.epb_payment_id+'/document', '_blank')`.

Card body:
- **Block 1 Vessel Ops:** render saat `epb.vessel_ops != null`. Field: Vessel, Crane, STS Slot, Mooring Team, ETA, Surveyor, Anchor, Est. Duration.
- **Block 2 Line Items:** render saat `epb.line_items.length > 0`. Columns: Item Layanan / Volume / Rate / Jumlah. Tfoot: Subtotal, PPn (label dinamis dari `vat_rate`), Total (bold + navy + tabular-nums).
- Currency formatter via helper `formatCurrency(amount, epb.currency)` — IDR pakai `id-ID`, USD pakai `en-US`.
- **Fallback minimal** (data legacy `vessel_ops` null + `line_items` kosong): render single Section Card sederhana — Nomor EPB / Total Tagihan / Currency / Due Date.

Action Card (right column kontextual) tetap:
- Tombol **"Bayar EPB"** → link ke `/customer/billing/epb/:epb_payment_id` (hanya tampil saat `payment_status = UNPAID`)

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
- [ ] Tombol "Bayar EPB" mengarah ke `/customer/billing/epb`
- [ ] Saat status `WAITING_PAYMENT_VERIFICATION`: tombol "Bayar EPB" disembunyikan, info banner teal tampil dengan link ke EPB & Invoice
- [ ] Tidak ada endpoint upload proof di M9 — upload dilakukan di M9b
- [ ] Revision form is only accessible when status is `NEED_REVISION`
- [ ] Re-submission sets status back to `PENDING`
- [ ] All nomination status changes recorded in `nomination_status_history`

### v2.2 additions (invoice-style EPB preview)
- [ ] Migration `nomination_epb` punya kolom extended: `subtotal`, `vat_rate`, `vat_amount`, `total_amount`, `currency`, `vessel_name`, `crane`, `sts_slot`, `mooring_team`, `eta`, `surveyor`, `anchor`, `est_duration_hours`, `bank_name`, `bank_account_no`, `bank_account_holder`, `kode_bayar`, `epb_pdf_url`.
- [ ] Migration tabel baru `nomination_epb_line_items` (epb_id, label, volume, rate, amount, sort_order).
- [ ] Webhook APPROVED handler insert seluruh field extended ke `nomination_epb` (NULL untuk field opsional yang tidak hadir di payload).
- [ ] Webhook APPROVED handler insert rows ke `nomination_epb_line_items` (jika ada).
- [ ] Response `GET /api/customer/nominations/:id/status` include `epb.vessel_ops`, `epb.line_items`, `epb.subtotal`, `epb.vat_*`, `epb.epb_pdf_url_available`, `epb.epb_payment_id`, `epb.payment_status`.
- [ ] Frontend EPB Detail section render Invoice Detail Card compact (Block 1 + Block 2) dari design system §3.2 v1.3.
- [ ] Tombol "Download EPB PDF" tampil di header EPB card; disabled saat `epb_pdf_url_available = false`.
- [ ] Tombol "Bayar EPB" navigate ke `/customer/billing/epb/:epb_payment_id` (specific EPB, bukan list).
- [ ] Fallback minimal untuk legacy nomination tanpa data extended — tidak boleh broken/kosong.
- [ ] Currency formatter konsisten dengan M9b (IDR id-ID, USD en-US).
