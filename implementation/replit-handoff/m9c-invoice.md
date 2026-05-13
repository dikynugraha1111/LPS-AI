# Replit Handoff — M9c: Invoice (Customer Portal)

**Version:** 1.0 (new module di v3.3)

> **Catatan:** Modul ini baru di BRD v3.3. Hasil pemisahan modul EPB & Invoice menjadi dua modul terpisah. M9c menangani **Invoice** (tagihan settlement). M9b menangani **EPB** (tagihan awal). UI keduanya tetap dalam satu menu "EPB & Invoice" di sidebar Customer Portal dengan tab internal terpisah.

## Context

**Module 9c** adalah modul Invoice di Customer Portal LPS. Mengelola **tagihan settlement** yang muncul dari:
1. **Shortfall EPB** — sisa pembayaran EPB yang dibayar parsial di M9b.
2. **Additional Service** — tagihan layanan tambahan (Tank Cleaning, Bunkering & Fresh Water Supplying, Short Stay Temporary, Supply Logistic, Lay Up, Ship Chandler, Kapal Emergency) yang dipakai customer selama voyage.

Invoice di-generate oleh STS Platform via webhook dan dikelola di LPS sebagai interface customer untuk upload bukti bayar.

### Key business rules
- Invoice **TIDAK** memblokir lifecycle nominasi. Status `nominations.status = PAYMENT_CONFIRMED` ditentukan oleh EPB Lunas (M9b), bukan oleh Invoice. Invoice = settlement separate.
- LPS **tidak** generate Invoice secara mandiri — semua dari webhook STS.
- **Tidak ada partial payment** di Invoice — harus dibayar penuh.
- Status lifecycle sama dengan EPB: Belum Dibayar → Menunggu Verifikasi → Pembayaran Ditolak (loop) atau Lunas (terminal).

### Data flow
1. STS Platform kirim webhook ke `/api/webhooks/sts/invoice`:
   - `EPB_SHORTFALL_DETECTED` (saat customer bayar EPB parsial dan STS confirm Lunas) — atau **forwarded** dari M9b handler (jika digabung dengan EPB_PAID).
   - `ADDITIONAL_SERVICE_INVOICE` (saat customer pakai service tambahan).
2. LPS create row di `invoices` dengan status `UNPAID`.
3. Customer buka tab Invoice → klik **"Bayar"** → upload proof → status berubah ke `WAITING_PAYMENT_VERIFICATION`.
4. STS verifies → kirim webhook `INVOICE_PAYMENT_REJECT` (loop) atau `INVOICE_PAID` (terminal).

## Prerequisites
- M7 complete: JWT auth, `customers` table.
- M8 complete: `nominations`, `nominations_additional_services` (m-to-m) tables.
- M9 complete: `nomination_epb` table.
- M9b complete: `epb_payments` table dengan kolom `shortfall_pushed`. M9b handler harus call M9c `invoiceService.CreateFromShortfall(...)` saat EPB_PAID dengan shortfall (atau pada event terpisah `EPB_SHORTFALL_DETECTED`).
- Env vars: `STS_WEBHOOK_SECRET`, `STS_API_KEY`, `STS_PLATFORM_BASE_URL`.

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — v1.2 (KPI Filter Pill, Overdue Date Display, Inline Info Banner, status labels v3.3).
2. **Per-modul UI design:** [`implementation/design/m9c-invoice-ui.md`](../design/m9c-invoice-ui.md) — list page dengan Source badge (Shortfall EPB / Additional Service) + KPI pills + tabs filter, detail page dengan source-specific section, cross-link ke parent EPB.
3. **Reference UI shared dengan M9b:** [`implementation/design/m9b-epb-invoice-ui.md`](../design/m9b-epb-invoice-ui.md) §3 (top tabs EPB/Invoice navigation).

**Surface:** A — Customer Portal (Bahasa Indonesia).

**UI rules ringkas:**
- Top tabs EPB | **Invoice** (shared dengan M9b — reuse `BillingLayout` component).
- KPI Filter Pills 3-up (Perlu Tindakan / Menunggu Verifikasi / Lunas).
- Tabs filter underline: Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas.
- **Additional filter Select** di kanan tabs: "Source — Semua / Shortfall EPB / Additional Service".
- **Source badge** (column baru di table):
  - `EPB_SHORTFALL` → "Shortfall EPB" (Neutral pill)
  - `ADDITIONAL_SERVICE` → "Additional Service" (Info pill)
- Status badge labels: Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas.
- Detail page: **Asal Tagihan section** dengan content source-specific:
  - Shortfall EPB → link ke parent EPB di M9b.
  - Additional Service → list service dengan label ID (mapping di `module/invoice/specifications.md` §9).
- **Bayar form: TIDAK ADA nominal input** — Invoice harus dibayar penuh.

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_invoices_m9c.up.sql`

```sql
CREATE TABLE IF NOT EXISTS invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number      VARCHAR(40) NOT NULL UNIQUE,
    customer_id         UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    nomination_id       UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    source              VARCHAR(30) NOT NULL,
    -- 'EPB_SHORTFALL' | 'ADDITIONAL_SERVICE'
    parent_epb_id       UUID REFERENCES epb_payments(id),
    service_keys        TEXT[],
    amount              NUMERIC(18,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'IDR',
    due_date            TIMESTAMPTZ,
    status              VARCHAR(40) NOT NULL DEFAULT 'UNPAID',
    -- 'UNPAID' | 'WAITING_PAYMENT_VERIFICATION' | 'PAYMENT_REJECT' | 'PAID'
    rejection_reason    TEXT,
    confirmed_at        TIMESTAMPTZ,
    sts_idempotency_key VARCHAR(255) UNIQUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT invoices_source_check CHECK (
        (source = 'EPB_SHORTFALL' AND parent_epb_id IS NOT NULL AND service_keys IS NULL) OR
        (source = 'ADDITIONAL_SERVICE' AND service_keys IS NOT NULL AND array_length(service_keys, 1) > 0)
    )
);

CREATE TABLE IF NOT EXISTS invoice_payment_proofs (
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

CREATE INDEX IF NOT EXISTS idx_invoices_customer_id      ON invoices(customer_id);
CREATE INDEX IF NOT EXISTS idx_invoices_nomination_id    ON invoices(nomination_id);
CREATE INDEX IF NOT EXISTS idx_invoices_status           ON invoices(status);
CREATE INDEX IF NOT EXISTS idx_invoices_source           ON invoices(source);
CREATE INDEX IF NOT EXISTS idx_invoices_due_date         ON invoices(due_date);
CREATE INDEX IF NOT EXISTS idx_invoices_parent_epb_id    ON invoices(parent_epb_id) WHERE parent_epb_id IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_invoice_proofs_invoice_id ON invoice_payment_proofs(invoice_id);
```

---

## Step 2: Go Models

File: `internal/invoice/model.go`

```go
type Invoice struct {
    ID                 uuid.UUID  `gorm:"type:uuid;primaryKey"`
    InvoiceNumber      string     `gorm:"size:40;not null;unique"`
    CustomerID         uuid.UUID  `gorm:"type:uuid;not null"`
    NominationID       uuid.UUID  `gorm:"type:uuid;not null"`
    Source             string     `gorm:"size:30;not null"`
    ParentEPBID        *uuid.UUID `gorm:"type:uuid"`
    ServiceKeys        pq.StringArray `gorm:"type:text[]"`
    Amount             decimal.Decimal `gorm:"type:numeric(18,2);not null"`
    Currency           string     `gorm:"size:3;not null;default:IDR"`
    DueDate            *time.Time
    Status             string     `gorm:"not null;default:UNPAID"`
    RejectionReason    *string
    ConfirmedAt        *time.Time
    STSIdempotencyKey  *string    `gorm:"size:255;uniqueIndex"`
    CreatedAt          time.Time
    UpdatedAt          time.Time

    ParentEPB *EPBPayment           `gorm:"foreignKey:ParentEPBID"`
    Proofs    []InvoicePaymentProof `gorm:"foreignKey:InvoiceID"`
}

type InvoicePaymentProof struct {
    ID              uuid.UUID `gorm:"type:uuid;primaryKey"`
    InvoiceID       uuid.UUID `gorm:"type:uuid;not null"`
    FileName        string    `gorm:"not null"`
    FileURL         string    `gorm:"not null"`
    FileSizeBytes   int       `gorm:"not null"`
    BankName        *string
    ReferenceNumber *string
    PaymentDate     *time.Time
    UploadedAt      time.Time `gorm:"default:now()"`
    STSSentAt       *time.Time
}

func (i *Invoice) IsOverdue() bool {
    if i.DueDate == nil || i.Status == "PAID" {
        return false
    }
    return i.DueDate.Before(time.Now())
}
```

Pakai `github.com/lib/pq` untuk `pq.StringArray` dan `github.com/shopspring/decimal`.

---

## Step 3: Backend — Internal Service (called by M9b)

File: `internal/invoice/service.go`

```go
type InvoiceService struct {
    db *gorm.DB
}

type CreateFromShortfallParams struct {
    LPSNominationID uuid.UUID
    ParentEPBID     uuid.UUID
    Amount          decimal.Decimal
    Currency        string
    DueDate         *time.Time
    IdempotencyKey  string
}

func (s *InvoiceService) CreateFromShortfall(params CreateFromShortfallParams) (*Invoice, error) {
    // Idempotency check
    var existing Invoice
    if err := s.db.Where("sts_idempotency_key = ?", params.IdempotencyKey).First(&existing).Error; err == nil {
        return &existing, nil // no-op
    }

    // Resolve customer_id from nomination
    var customerID uuid.UUID
    if err := s.db.Model(&Nomination{}).Where("id = ?", params.LPSNominationID).Select("customer_id").Scan(&customerID).Error; err != nil {
        return nil, err
    }

    inv := Invoice{
        InvoiceNumber:     generateInvoiceNumber(), // INV-YYYYMMDD-NNNNN
        CustomerID:        customerID,
        NominationID:      params.LPSNominationID,
        Source:            "EPB_SHORTFALL",
        ParentEPBID:       &params.ParentEPBID,
        Amount:            params.Amount,
        Currency:          params.Currency,
        DueDate:           params.DueDate,
        Status:            "UNPAID",
        STSIdempotencyKey: &params.IdempotencyKey,
    }
    if err := s.db.Create(&inv).Error; err != nil {
        return nil, err
    }

    // Send notification to customer
    notify(customerID, "Invoice baru: Sisa pembayaran Rp "+inv.Amount.String()+" telah ditagihkan.")

    return &inv, nil
}

type CreateFromAdditionalServiceParams struct {
    LPSNominationID uuid.UUID
    ServiceKeys     []string
    Amount          decimal.Decimal
    Currency        string
    DueDate         *time.Time
    IdempotencyKey  string
}

func (s *InvoiceService) CreateFromAdditionalService(params CreateFromAdditionalServiceParams) (*Invoice, error) {
    // Similar pattern; source = 'ADDITIONAL_SERVICE', service_keys = params.ServiceKeys
    // ...
}
```

---

## Step 4: Backend — STS Webhook Handler

File: `internal/webhook/sts_invoice_handler.go`

### POST /api/webhooks/sts/invoice

HMAC verification (reuse helper).

**Payload:**
```go
type STSInvoiceWebhookPayload struct {
    LPSNominationID *string         `json:"lps_nomination_id"` // untuk CREATE events
    LPSInvoiceID    *string         `json:"lps_invoice_id"`    // untuk UPDATE events
    Event           string          `json:"event"`
    Timestamp       time.Time       `json:"timestamp"`
    Data            json.RawMessage `json:"data"`
}
```

Route by `Event`:

**Event: `EPB_SHORTFALL_DETECTED`** (jika dikirim STS langsung ke endpoint ini, bukan via M9b forward)
```go
type ShortfallData struct {
    InvoiceNumber  string          `json:"invoice_number"`
    ParentEPBNumber string         `json:"parent_epb_number"`
    Amount         decimal.Decimal `json:"amount"`
    Currency       string          `json:"currency"`
    DueDate        *time.Time      `json:"due_date"`
    IdempotencyKey string          `json:"idempotency_key"`
}
```
Actions:
1. Resolve `parent_epb_id` dari `parent_epb_number`.
2. Call `invoiceService.CreateFromShortfall(...)`.
3. Return 200.

**Event: `ADDITIONAL_SERVICE_INVOICE`**
```go
type AddServiceData struct {
    InvoiceNumber  string          `json:"invoice_number"`
    ServiceKeys    []string        `json:"service_keys"`
    Amount         decimal.Decimal `json:"amount"`
    Currency       string          `json:"currency"`
    DueDate        *time.Time      `json:"due_date"`
    IdempotencyKey string          `json:"idempotency_key"`
}
```
Actions:
1. Call `invoiceService.CreateFromAdditionalService(...)`.
2. Return 200.

**Event: `INVOICE_PAYMENT_REJECT`**
```go
type RejectData struct {
    RejectionReason string `json:"rejection_reason"`
}
```
Actions:
1. Find `invoices` by `id = LPSInvoiceID`.
2. Update `status = 'PAYMENT_REJECT'`, `rejection_reason = data.rejection_reason`.
3. Send notification.

**Event: `INVOICE_PAID`**
```go
type PaidData struct {
    ConfirmedAt time.Time `json:"confirmed_at"`
}
```
Actions:
1. Find `invoices` by `id = LPSInvoiceID`.
2. Update `status = 'PAID'`, `confirmed_at = data.confirmed_at`.
3. Send notification.
4. **Important:** TIDAK mengubah `nominations.status` (Invoice tidak block nominasi).

---

## Step 5: Backend — List & Detail Invoice

File: `internal/invoice/handler.go`

### GET /api/customer/invoices

Auth: Customer JWT.

Query params:
- `status` (optional)
- `source` (optional): `EPB_SHORTFALL` | `ADDITIONAL_SERVICE`
- `tab` (optional)

SQL:
```sql
SELECT i.*, n.nomination_number,
       ep.id as parent_epb_id, ne.epb_number as parent_epb_number,
       (SELECT row_to_json(p) FROM invoice_payment_proofs p
        WHERE p.invoice_id = i.id ORDER BY p.uploaded_at DESC LIMIT 1) AS latest_proof
FROM invoices i
JOIN nominations n ON i.nomination_id = n.id
LEFT JOIN epb_payments ep ON i.parent_epb_id = ep.id
LEFT JOIN nomination_epb ne ON ep.epb_id = ne.id
WHERE i.customer_id = $1
ORDER BY i.updated_at DESC
```

### GET /api/customer/invoices/summary

Same pattern as M9b summary.

### GET /api/customer/invoices/:id

Full detail termasuk parent EPB info (jika source=EPB_SHORTFALL) atau service labels (jika source=ADDITIONAL_SERVICE).
Guard: customer_id check.

---

## Step 6: Backend — Upload Payment Proof

### POST /api/customer/invoices/:id/proof

Auth: Customer JWT.

Multipart form fields:
- `file` (required) — PDF/JPG/PNG, max 5MB
- `bank_name` (optional)
- `reference_number` (optional)
- `payment_date` (optional)

> Tidak ada `paid_amount` — Invoice harus dibayar penuh.

Validation:
1. `invoice.customer_id == JWT customer ID` (FAIL → 403).
2. `invoice.status` must be `UNPAID` or `PAYMENT_REJECT` (FAIL → 409).
3. File MIME + size validation.

Logic (transaction):
1. Save file: `uploads/invoices/:invoice_id/:timestamp_filename`.
2. Insert `invoice_payment_proofs` row.
3. Update `invoices.status = 'WAITING_PAYMENT_VERIFICATION'`, `rejection_reason = NULL`, `updated_at = now()`.
4. **TIDAK mengubah `nominations.status`** (Invoice tidak block nominasi).
5. Commit.
6. Async: send proof to STS Platform (Step 7).
7. Return 201.

---

## Step 7: Backend — Send Proof to STS Platform

```
POST {STS_PLATFORM_BASE_URL}/api/invoices/{invoice_number}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body (JSON):
{
  "lps_invoice_id": "uuid",
  "invoice_number": "INV-20260505-00001",
  "amount": 5000000.00,
  "currency": "IDR",
  "file_url": "https://storage.lps.internal/...",
  "file_name": "bukti_bayar_invoice.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-789012",
  "payment_date": "2026-05-10",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff.

---

## Step 8: Frontend — Invoice List Page

File: `src/pages/customer/billing/InvoiceListPage.tsx`

Layout reuses `BillingLayout` dari M9b (top tabs EPB | **Invoice**).

Components:
- **KPI Filter Pills** (3-up): query `/api/customer/invoices/summary`.
- **Tabs filter underline:** Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas.
- **Source filter Select** (kanan tabs): "Semua / Shortfall EPB / Additional Service".
- **Data Table:** No. Invoice, **Source badge**, Ref Nominasi, Amount, Jatuh Tempo (dengan Overdue display), Status badge, Aksi.

Status mapping helper (sama dengan M9b):
```ts
const STATUS_LABELS = {
  UNPAID: { label: "Belum Dibayar", variant: "neutral" },
  WAITING_PAYMENT_VERIFICATION: { label: "Menunggu Verifikasi", variant: "pendingVerify" },
  PAYMENT_REJECT: { label: "Pembayaran Ditolak", variant: "error" },
  PAID: { label: "Lunas", variant: "confirmed" },
};

const SOURCE_LABELS = {
  EPB_SHORTFALL: { label: "Shortfall EPB", variant: "neutral" },
  ADDITIONAL_SERVICE: { label: "Additional Service", variant: "info" },
};
```

Action column untuk row `UNPAID`: tombol "Bayar" + link "Lihat Detail →".

---

## Step 9: Frontend — Invoice Detail Page

File: `src/pages/customer/billing/InvoiceDetailPage.tsx`

Layout 2-column (lihat m9c-invoice-ui.md §4).

**Asal Tagihan Section** (source-specific):

```tsx
{invoice.source === "EPB_SHORTFALL" && (
  <div>
    <div className="text-xs uppercase tracking-wider text-slate-500 mb-2">Asal Tagihan</div>
    <div className="text-base font-semibold text-slate-900">Shortfall EPB</div>
    <p className="text-sm text-slate-600 mt-2">
      Tagihan ini muncul karena pembayaran EPB sebelumnya tidak penuh.
    </p>
    <Link href={`/customer/billing/epb/${invoice.parent_epb_id}`} className="...">
      {invoice.parent_epb_number} <ArrowRight className="h-3.5 w-3.5" />
    </Link>
  </div>
)}

{invoice.source === "ADDITIONAL_SERVICE" && (
  <div>
    <div className="text-xs uppercase tracking-wider text-slate-500 mb-2">Asal Tagihan</div>
    <div className="text-base font-semibold text-slate-900">Additional Service</div>
    <p className="text-sm text-slate-600 mt-2">
      Tagihan ini muncul dari layanan tambahan yang Anda gunakan selama voyage.
    </p>
    <ul className="space-y-1.5 mt-3">
      {invoice.service_keys.map(key => (
        <li key={key} className="flex items-center gap-2 text-sm text-slate-700">
          <Check className="h-4 w-4 text-emerald-600" />
          {SERVICE_LABELS[key]}
        </li>
      ))}
    </ul>
  </div>
)}
```

`SERVICE_LABELS` constant:
```ts
const SERVICE_LABELS: Record<string, string> = {
  tank_cleaning: "Tank Cleaning",
  bunkering: "Pengisian Bahan Bakar atau Air Bersih",
  short_stay_temporary: "Short Stay Temporary",
  supply_logistic: "Supply Logistic",
  lay_up: "Lay Up",
  ship_chandler: "Ship Chandler",
  kapal_emergency: "Kapal Emergency",
};
```

**Bayar Form:**
- **TIDAK ada nominal input** — display amount read-only ("Total tagihan: Rp {amount}").
- File upload + optional bank fields.
- Submit → status berubah ke Menunggu Verifikasi; action card hide.

---

## Acceptance Checklist

### Backend
- [ ] Migration `invoices` table dengan source constraint check (EPB_SHORTFALL requires parent_epb_id; ADDITIONAL_SERVICE requires service_keys).
- [ ] `InvoiceService.CreateFromShortfall(...)` callable dari M9b webhook handler dengan idempotency.
- [ ] `InvoiceService.CreateFromAdditionalService(...)` callable dari STS webhook ADDITIONAL_SERVICE_INVOICE.
- [ ] `POST /api/webhooks/sts/invoice` HMAC verified, route by event (4 events).
- [ ] Idempotency check pakai `sts_idempotency_key` — duplicate webhook tidak buat row baru.
- [ ] `INVOICE_PAID` **TIDAK** mengubah `nominations.status` (Invoice tidak block nominasi).
- [ ] `POST /api/customer/invoices/:id/proof` validate status, save file, update status ke WAITING_PAYMENT_VERIFICATION, async send to STS.
- [ ] `GET /api/customer/invoices/summary` returns 3 counts.

### Frontend
- [ ] Top tabs EPB | Invoice (reuse BillingLayout). Klik "Invoice" → list page.
- [ ] KPI Pills 3-up + Tabs filter + Source filter Select.
- [ ] Source badges: "Shortfall EPB" neutral / "Additional Service" info.
- [ ] Status badge labels v3.3 (Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas).
- [ ] Overdue indicator pada due_date < now() dengan status != Lunas.
- [ ] Detail page Asal Tagihan section conditional render:
  - EPB_SHORTFALL → link ke parent EPB di M9b.
  - ADDITIONAL_SERVICE → list service dengan label ID.
- [ ] Bayar form TIDAK menampilkan input nominal — Invoice full payment.
- [ ] Submit Bayar → status berubah ke Menunggu Verifikasi.
- [ ] Bank account card dengan copy-to-clipboard.
- [ ] Riwayat Pembayaran nested cards per attempt.
- [ ] Empty state list kosong: "Belum ada Invoice. Akan muncul jika ada shortfall EPB atau Additional Service."
