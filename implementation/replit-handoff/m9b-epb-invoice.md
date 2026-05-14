# Replit Handoff — M9b: EPB (Customer Portal)

**Version:** 2.2 (v3.5 — Detail Nominasi section + Payment Instruction Box exact spec)

> **Version history:**
> - **v1.0** — 2026-05-07: Initial M9b handoff (gabungan EPB & Invoice scope, no partial payment).
> - **v2.0** — 2026-05-13: Split M9b → EPB only. Invoice (settlement) pindah ke M9c. Tambah partial payment (min 1 USD ekuivalen IDR). Status labels disinkronkan ke Bahasa Indonesia natural. Route berubah dari `/customer/epb-invoice` → `/customer/billing/epb`. Webhook `EPB_SHORTFALL_DETECTED` ditambah untuk trigger Invoice creation di M9c.
> - **v2.1** — 2026-05-14: Detail Tagihan diupgrade ke **invoice-style** (Invoice Detail Card dari design system §3.2 v1.3 — Vessel Ops Grid + Line Items Table + Payment Instruction Box). Data tambahan dibaca dari `nomination_epb` (extended di M9 v2.1) lewat JOIN di endpoint detail. Tombol **Download EPB PDF** di header (FR-EI-12) — proxy ke STS via `GET /api/customer/epb-payments/:id/document`. Multi-currency support (IDR/USD) via `Intl.NumberFormat`. Fallback minimal untuk EPB legacy tanpa data extended.
> - **v2.2** — 2026-05-14: Tambah section **"Detail Nominasi"** di halaman EPB detail (FR-EI-13) — selalu tampil, menampilkan data nominasi induk (Kapal, Tipe Kapal, Jenis Cargo, Towage Plan, ETA, Agen, Charterer, Dibuat, Diperbarui) dari JOIN `nominations`. API endpoint `GET /api/customer/epb-payments/:id` diperluas dengan field `nomination_data`. Spesifikasi Block 3 (Payment Instruction Box) dipertegas: 2-column grid, Total bold + navy.

## Context

Continuation of the LPS Customer Portal. **Module 9b** sekarang adalah modul **EPB (Estimasi Perkiraan Biaya)** — tagihan awal yang diterbitkan STS Platform setelah nominasi disetujui. Sejak v3.3, scope M9b adalah **EPB only**. Bagian Invoice (settlement) dipisah ke M9c.

Customer dapat membayar EPB **penuh atau parsial** (minimum 1 USD ekuivalen IDR). Saat EPB ter-Lunas dengan pembayaran parsial, STS Platform mengirim webhook `EPB_SHORTFALL_DETECTED` yang diteruskan ke handler M9c untuk membuat row Invoice baru.

UI berada di tab **"EPB"** dalam menu "EPB & Invoice" di sidebar Customer Portal (sidebar menu tetap satu — sub-tab EPB / Invoice di dalam halaman).

### Data flow
1. M9 webhook handler (event `APPROVED`) → otomatis buat `epb_payments` row dengan status `UNPAID` (label UI: "Belum Dibayar") tanpa aksi customer.
2. Customer buka tab EPB, lihat EPB berstatus Belum Dibayar, klik **"Bayar"** → **input nominal** (≥ 1 USD ekuivalen IDR) → upload proof → submit → status berubah ke `WAITING_PAYMENT_VERIFICATION` (label UI: "Menunggu Verifikasi") di `epb_payments` DAN `nominations`.
3. STS verifies → kirim webhook `EPB_PAYMENT_REJECT` (loop ke step 2 via Revisi Data) atau `EPB_PAID` (terminal).
4. Jika `EPB_PAID` dengan `paid_amount < total_amount`: STS kirim webhook `EPB_SHORTFALL_DETECTED` → handler M9b meneruskan ke M9c untuk create Invoice row baru.

## Prerequisites
- M7 complete: JWT auth, `customers` table.
- M8 complete: `nominations`, `nomination_documents` tables.
- M9 complete: `nomination_epb`, `nomination_status_history` tables; STS webhook `APPROVED` handler berfungsi dan otomatis membuat row `epb_payments` (status `UNPAID`) untuk setiap nominasi yang disetujui.
- M9c skeleton akan dibuat di handoff `m9c-invoice.md` — handler `EPB_SHORTFALL_DETECTED` di M9b memforward call ke M9c create function.
- System Config `USD_IDR_RATE` (refresh harian) + `MIN_PAYMENT_IDR_FALLBACK` (default `15000`).
- Env vars: `STS_WEBHOOK_SECRET`, `STS_API_KEY`, `STS_PLATFORM_BASE_URL`.

## Prerequisites — Design Reference (WAJIB)

Sebelum menulis UI apapun, **wajib** baca:

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — v1.3 (Invoice Detail Card + Line Items Table + Payment Instruction Box, multi-currency).
2. **Per-modul UI design:** [`implementation/design/m9b-epb-invoice-ui.md`](../design/m9b-epb-invoice-ui.md) — list page dengan KPI pills + tabs filter, **detail page pakai Invoice Detail Card (3 blok: Vessel Ops Grid + Line Items Table + Payment Instruction Box)**, status banner per 4 payment status, **Bayar form dengan nominal input + validasi partial payment**, **tombol Download EPB PDF di header**, cross-link ke Invoice saat shortfall.

**Surface:** A — Customer Portal (Bahasa Indonesia).

**UI rules ringkas v3.3:**
- Tabs underline besar di top of page: **EPB** | Invoice (primary navigation).
- KPI Filter Pills 3-up (Perlu Tindakan rose / Menunggu Verifikasi amber / Lunas emerald) — clickable shortcut filter.
- Tabs filter underline: Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas.
- **Status badge labels v3.3:**
  - `UNPAID` → "Belum Dibayar" (Neutral pill)
  - `WAITING_PAYMENT_VERIFICATION` → "Menunggu Verifikasi" (Pending verify pill)
  - `PAYMENT_REJECT` → "Pembayaran Ditolak" (Error pill)
  - `PAID` → "Lunas" (Confirmed pill)
- Overdue Date Display: `text-rose-700 + icon AlertTriangle` saat `due_date < now()` AND status != Lunas.
- **Bayar form** dengan field nominal pembayaran (numeric IDR, default = total_amount, editable), helper text min/max, inline validation.
- Bank account card dengan tombol copy-to-clipboard.
- Payment history nested cards per attempt dengan `paid_amount` snapshot + reason saat reject.
- File upload dropzone: design system §3.2.
- Cross-link ke Invoice saat EPB Lunas dengan shortfall (info banner amber + link).

## Tech Stack
- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration

File: `migrations/XXXXXX_create_epb_payments_m9b.up.sql`

**Catatan migrasi:** Jika tabel `epb_payments` sudah ada dari v1.0, gunakan `ALTER TABLE` untuk menambah kolom baru. Cek struktur yang sudah ada sebelum jalankan.

```sql
-- ⚠️ v2.0 change: tambah kolom total_amount, paid_amount, currency, due_date, shortfall_pushed
-- Jika tabel sudah ada (v1.0): pakai ALTER TABLE.
-- Jika fresh: CREATE TABLE penuh di bawah.

CREATE TABLE IF NOT EXISTS epb_payments (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id     UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_id            UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    status            VARCHAR(40) NOT NULL DEFAULT 'UNPAID',
    -- 'UNPAID' | 'WAITING_PAYMENT_VERIFICATION' | 'PAYMENT_REJECT' | 'PAID'
    total_amount      NUMERIC(18,2) NOT NULL,
    paid_amount       NUMERIC(18,2),
    currency          VARCHAR(3) NOT NULL DEFAULT 'IDR',
    due_date          TIMESTAMPTZ,
    rejection_reason  TEXT,
    confirmed_at      TIMESTAMPTZ,
    shortfall_pushed  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS epb_payment_proofs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epb_payment_id   UUID NOT NULL REFERENCES epb_payments(id) ON DELETE CASCADE,
    paid_amount      NUMERIC(18,2) NOT NULL,
    file_name        VARCHAR(255) NOT NULL,
    file_url         TEXT NOT NULL,
    file_size_bytes  INTEGER NOT NULL,
    bank_name        VARCHAR(100),
    reference_number VARCHAR(100),
    payment_date     DATE,
    uploaded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sts_sent_at      TIMESTAMPTZ
);

CREATE INDEX IF NOT EXISTS idx_epb_payments_nomination_id ON epb_payments(nomination_id);
CREATE INDEX IF NOT EXISTS idx_epb_payments_status        ON epb_payments(status);
CREATE INDEX IF NOT EXISTS idx_epb_payments_due_date      ON epb_payments(due_date);
CREATE INDEX IF NOT EXISTS idx_epb_proofs_payment_id      ON epb_payment_proofs(epb_payment_id);

-- ⚠️ Update M9 webhook handler (event APPROVED): insert row dengan total_amount dari STS EPB data.
-- Sample insert dari M9 handler:
-- INSERT INTO epb_payments (nomination_id, epb_id, status, total_amount, currency, due_date)
-- VALUES ($1, $2, 'UNPAID', $3, 'IDR', $4);
```

---

## Step 2: Go Models

File: `internal/epbpayment/model.go`

```go
type EPBPayment struct {
    ID               uuid.UUID  `gorm:"type:uuid;primaryKey"`
    NominationID     uuid.UUID  `gorm:"type:uuid;not null"`
    EPBID            uuid.UUID  `gorm:"type:uuid;not null"`
    Status           string     `gorm:"not null;default:UNPAID"`
    TotalAmount      decimal.Decimal `gorm:"type:numeric(18,2);not null"`
    PaidAmount       *decimal.Decimal `gorm:"type:numeric(18,2)"`
    Currency         string     `gorm:"size:3;not null;default:IDR"`
    DueDate          *time.Time
    RejectionReason  *string
    ConfirmedAt      *time.Time
    ShortfallPushed  bool       `gorm:"not null;default:false"`
    CreatedAt        time.Time
    UpdatedAt        time.Time

    EPB    NominationEPB     `gorm:"foreignKey:EPBID"`
    Proofs []EPBPaymentProof `gorm:"foreignKey:EPBPaymentID"`
}

type EPBPaymentProof struct {
    ID              uuid.UUID `gorm:"type:uuid;primaryKey"`
    EPBPaymentID    uuid.UUID `gorm:"type:uuid;not null"`
    PaidAmount      decimal.Decimal `gorm:"type:numeric(18,2);not null"`
    FileName        string    `gorm:"not null"`
    FileURL         string    `gorm:"not null"`
    FileSizeBytes   int       `gorm:"not null"`
    BankName        *string
    ReferenceNumber *string
    PaymentDate     *time.Time
    UploadedAt      time.Time `gorm:"default:now()"`
    STSSentAt       *time.Time
}

// IsOverdue computed property — used by API serializer.
func (e *EPBPayment) IsOverdue() bool {
    if e.DueDate == nil || e.Status == "PAID" {
        return false
    }
    return e.DueDate.Before(time.Now())
}
```

Pakai `github.com/shopspring/decimal` untuk monetary precision.

---

## Step 3: Backend — STS Webhook Handler

File: `internal/webhook/sts_epb_handler.go`

### POST /api/webhooks/sts/epb-payment-status

HMAC verification via `verifySTSSignature` (reuse dari M9).

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

**Event: `EPB_PAYMENT_REJECT`**
```go
type EPBPaymentRejectData struct {
    RejectionReason string `json:"rejection_reason"`
}
```
Actions:
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Update `status = 'PAYMENT_REJECT'`, `rejection_reason = data.rejection_reason`, `updated_at = now()`.
3. Send notification: "Bukti pembayaran ditolak: [rejection_reason]. Silakan upload ulang."
4. Return 200 OK.

**Event: `EPB_PAID`**
```go
type EPBPaidData struct {
    ConfirmedAt     time.Time       `json:"confirmed_at"`
    PaidAmount      decimal.Decimal `json:"paid_amount"`
    ShortfallAmount decimal.Decimal `json:"shortfall_amount"` // 0 atau positif
}
```
Actions (dalam satu transaksi DB):
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Update `epb_payments`: `status = 'PAID'`, `paid_amount = data.paid_amount`, `confirmed_at = data.confirmed_at`.
3. Update `nominations.status = 'PAYMENT_CONFIRMED'` (paralel; voyage bisa lanjut).
4. **Jika `data.shortfall_amount > 0`:**
   - Call M9c handler: `invoiceService.CreateFromShortfall(epb_payment_id, shortfall_amount, due_date)`.
   - Set `epb_payments.shortfall_pushed = TRUE`.
5. Notifications:
   - Tanpa shortfall: "Pembayaran EPB dikonfirmasi. Voyage dapat dimulai."
   - Dengan shortfall: "Pembayaran EPB dikonfirmasi. Sisa pembayaran Rp {shortfall} telah ditagihkan via Invoice. Silakan cek tab Invoice."
6. Return 200 OK.

**Event: `EPB_SHORTFALL_DETECTED`** (idempotency-safe; bisa dikirim terpisah dari EPB_PAID jika STS update belakangan)
```go
type EPBShortfallData struct {
    ShortfallAmount decimal.Decimal `json:"shortfall_amount"`
    InvoiceDueDate  *time.Time      `json:"invoice_due_date"`
}
```
Actions:
1. Find `epb_payments`.
2. Idempotency check: jika `shortfall_pushed = TRUE`, return 200 (no-op).
3. Call M9c handler: `invoiceService.CreateFromShortfall(...)`.
4. Set `epb_payments.shortfall_pushed = TRUE`.
5. Return 200 OK.

---

## Step 4: Backend — List & Detail EPB

File: `internal/epbpayment/handler.go`

### GET /api/customer/epb-payments

Auth: Customer JWT.

Query params:
- `status` (optional): single status atau `action_required` (= UNPAID + PAYMENT_REJECT + Overdue)
- `tab` (optional): same effect as status filter, more UX-friendly value

SQL:
```sql
SELECT ep.*, ne.epb_number, n.nomination_number, n.customer_id,
       (SELECT row_to_json(p) FROM epb_payment_proofs p
        WHERE p.epb_payment_id = ep.id ORDER BY p.uploaded_at DESC LIMIT 1) AS latest_proof
FROM epb_payments ep
JOIN nominations n ON ep.nomination_id = n.id
JOIN nomination_epb ne ON ep.epb_id = ne.id
WHERE n.customer_id = $1
ORDER BY ep.updated_at DESC
```

Serialize dengan `is_overdue` computed field.

### GET /api/customer/epb-payments/summary

```go
type SummaryResponse struct {
    ActionRequired      int `json:"action_required"`
    WaitingVerification int `json:"waiting_verification"`
    Paid                int `json:"paid"`
}
```

SQL (single query):
```sql
SELECT
  COUNT(*) FILTER (WHERE ep.status = 'UNPAID' OR ep.status = 'PAYMENT_REJECT' OR (ep.due_date < now() AND ep.status != 'PAID')) AS action_required,
  COUNT(*) FILTER (WHERE ep.status = 'WAITING_PAYMENT_VERIFICATION') AS waiting_verification,
  COUNT(*) FILTER (WHERE ep.status = 'PAID') AS paid
FROM epb_payments ep
JOIN nominations n ON ep.nomination_id = n.id
WHERE n.customer_id = $1
```

### GET /api/customer/epb-payments/:id

Full detail: EPB record + all proof attempts (DESC by uploaded_at) + nomination data + invoice-style data.
Guard: customer_id check via JOIN to nominations.

SQL (v2.2 — tiga-tabel JOIN):
```sql
SELECT
  ep.*,
  ne.epb_number, ne.vessel_name AS ops_vessel, ne.crane, ne.sts_slot,
  ne.mooring_team, ne.eta AS ops_eta, ne.surveyor, ne.anchor,
  ne.est_duration_hours, ne.subtotal, ne.vat_rate, ne.vat_amount,
  ne.bank_name, ne.bank_account_no, ne.bank_account_holder, ne.kode_bayar,
  ne.epb_pdf_url,
  n.nomination_number, n.customer_id,
  -- nomination_data fields (v2.2)
  n.vessel_name,
  n.vessel_type,
  n.cargo_type,
  n.towage_plan,
  n.eta           AS nom_eta,
  n.agent_name,
  n.charterer,
  n.created_at    AS nom_created_at,
  n.updated_at    AS nom_updated_at
FROM epb_payments ep
JOIN nomination_epb ne ON ep.epb_id = ne.id
JOIN nominations n ON ep.nomination_id = n.id
WHERE ep.id = $1 AND n.customer_id = $2
```

Serialize `nomination_data` sebagai nested object di response JSON. `nomination_epb_line_items` di-fetch terpisah (SELECT WHERE epb_id = ne.id).

---

## Step 5: Backend — Upload Payment Proof (with Partial Payment)

### POST /api/customer/epb-payments/:id/proof

Auth: Customer JWT.

Multipart form fields:
- `file` (required) — PDF/JPG/PNG, max 5MB
- `paid_amount` (required) — string decimal, parse to `decimal.Decimal`
- `bank_name` (optional)
- `reference_number` (optional)
- `payment_date` (optional, format YYYY-MM-DD)

Validation:
1. `epb_payment.nomination.customer_id == JWT customer ID` (FAIL → 403).
2. `epb_payment.status` must be `UNPAID` or `PAYMENT_REJECT` (FAIL → 409 Conflict).
3. File MIME validation + size.
4. **Compute min IDR:**
   ```go
   usdRate := configService.GetFloat("USD_IDR_RATE", 0)
   var minIDR decimal.Decimal
   if usdRate > 0 {
       minIDR = decimal.NewFromFloat(usdRate) // 1 USD * rate
   } else {
       minIDR = decimal.NewFromInt(configService.GetInt("MIN_PAYMENT_IDR_FALLBACK", 15000))
   }
   ```
5. Validate `paid_amount`:
   - `>= minIDR` (FAIL → 400 "Nominal kurang dari minimum 1 USD.")
   - `<= epb_payment.total_amount` (FAIL → 400 "Nominal melebihi total tagihan.")

Logic (transaction):
1. Save file: `uploads/epb-payments/:epb_payment_id/:timestamp_filename`.
2. Insert `epb_payment_proofs` row (with `paid_amount` snapshot).
3. Update `epb_payments`: `status = 'WAITING_PAYMENT_VERIFICATION'`, `paid_amount = <input>`, `rejection_reason = NULL`, `updated_at = now()`.
4. Update `nominations.status = 'WAITING_PAYMENT_VERIFICATION'` (single transaction).
5. Commit.
6. Async: send proof to STS Platform (Step 6).
7. Return 201 with updated payment state.

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
  "paid_amount": 10000000.00,
  "currency": "IDR",
  "file_url": "https://storage.lps.internal/...",
  "file_name": "bukti_bayar.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-123456",
  "payment_date": "2026-05-01",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff (0.5s, 1s, 2s). On all retries fail: log error; status di LPS tetap `WAITING_PAYMENT_VERIFICATION` (jangan surface error ke customer).

On success: update `epb_payment_proofs.sts_sent_at = now()`.

---

## Step 7: Frontend — Billing Landing & Tab Navigation

File: `src/pages/customer/billing/BillingLayout.tsx`

Pattern shared dengan M9c. Layout:

```tsx
import { useLocation, Link, Route, Switch, Redirect } from "wouter";

export function BillingLayout() {
  const [location] = useLocation();
  const isEPB = location.startsWith("/customer/billing/epb");
  const isInvoice = location.startsWith("/customer/billing/invoice");

  return (
    <CustomerLayout>
      <PageHeader
        icon={<Receipt className="h-6 w-6 text-slate-500" />}
        title="EPB & Invoice"
        subtitle="Kelola dan lacak status pembayaran EPB & Invoice nominasi Anda."
      />

      {/* Primary tabs */}
      <div className="border-b border-slate-200 flex items-center gap-8 mb-6">
        <Link href="/customer/billing/epb"
              className={isEPB ? "border-b-2 border-[#0F2A4D] text-slate-900 font-semibold pb-3 text-base" : "text-slate-500 hover:text-slate-900 pb-3 text-base"}>
          EPB
        </Link>
        <Link href="/customer/billing/invoice"
              className={isInvoice ? "border-b-2 border-[#0F2A4D] text-slate-900 font-semibold pb-3 text-base" : "text-slate-500 hover:text-slate-900 pb-3 text-base"}>
          Invoice
        </Link>
      </div>

      {/* Tab content */}
      <Switch>
        <Route path="/customer/billing" component={() => <Redirect to="/customer/billing/epb" />} />
        <Route path="/customer/billing/epb" component={EPBListPage} />
        <Route path="/customer/billing/epb/:id" component={EPBDetailPage} />
        <Route path="/customer/billing/invoice" component={InvoiceListPage} />
        <Route path="/customer/billing/invoice/:id" component={InvoiceDetailPage} />
      </Switch>
    </CustomerLayout>
  );
}
```

Sidebar nav update: item "EPB & Invoice" link ke `/customer/billing`.

---

## Step 8: Frontend — EPB List Page

File: `src/pages/customer/billing/EPBListPage.tsx`

Components:
- **KPI Filter Pills row** (3 cards): pakai `useQuery` ke `/api/customer/epb-payments/summary`. Klik pill → set `tab` query state.
- **Tabs filter** (underline): Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas. Active tab di-control via state.
- **Data Table:** kolom No. EPB, Ref Nominasi, Total, Jatuh Tempo (dengan Overdue display jika `is_overdue`), Status badge, Aksi.

Status mapping helper:
```ts
const STATUS_LABELS: Record<string, { label: string; variant: BadgeVariant }> = {
  UNPAID: { label: "Belum Dibayar", variant: "neutral" },
  WAITING_PAYMENT_VERIFICATION: { label: "Menunggu Verifikasi", variant: "pendingVerify" },
  PAYMENT_REJECT: { label: "Pembayaran Ditolak", variant: "error" },
  PAID: { label: "Lunas", variant: "confirmed" },
};
```

Filter logic:
```ts
const filterByTab = (items, tab) => {
  if (tab === "all") return items;
  if (tab === "action_required") return items.filter(i =>
    i.status === "UNPAID" || i.status === "PAYMENT_REJECT" || (i.is_overdue && i.status !== "PAID"));
  if (tab === "waiting_verification") return items.filter(i => i.status === "WAITING_PAYMENT_VERIFICATION");
  if (tab === "paid") return items.filter(i => i.status === "PAID");
};
```

Action column conditional:
```tsx
{item.status === "UNPAID" && (
  <Button size="sm" onClick={() => navigate(`/customer/billing/epb/${item.id}?action=bayar`)}>
    Bayar
  </Button>
)}
<Link className="link-action">Lihat Detail <ArrowRight /></Link>
```

---

## Step 9: Frontend — EPB Detail Page (v2.1 invoice-style)

File: `src/pages/customer/billing/EPBDetailPage.tsx`

Layout 2-column (lihat m9b-epb-invoice-ui.md §5).

**Detail data dari API** — `GET /api/customer/epb-payments/:id` mereturn payload extended (JOIN `nomination_epb` + `nominations`):
- Identitas: `epb_number`, `nomination_number`, `status`, `total_amount`, `paid_amount`, `currency` (`IDR`/`USD`), `due_date`.
- **Nomination data (v2.2, dari JOIN `nominations`):**
  - `nomination_data` — `{ vessel_name, vessel_type, cargo_type, towage_plan, eta, agent_name, charterer, created_at, updated_at }`
  - Nullable jika legacy; UI hide section kalau null.
- Invoice-style data (optional, dari `nomination_epb`):
  - `vessel_ops` — `{ vessel_name, crane, sts_slot, mooring_team, eta, surveyor, anchor, est_duration_hours }`
  - `line_items[]` — `[{ label, volume, rate, amount }]`
  - `subtotal`, `vat_rate`, `vat_amount`
  - `bank_info` — `{ bank_name, account_number, account_holder, kode_bayar }`
  - `epb_pdf_url`
- `proofs[]` — riwayat upload attempt.

Key sections:
- **Header bar:**
  - Kiri: `EPB-{epb_number}` (text-2xl font-bold mono) + sub-text `Ref Nominasi {nomination_number}` muted.
  - Kanan: `<StatusBadge>` + tombol `<Button variant="outline"><Download /> Download EPB PDF</Button>`.
  - Download click: `window.open('/api/customer/epb-payments/'+id+'/document', '_blank')`. Jika `epb_pdf_url` null, disable tombol + tooltip "Dokumen sedang dipersiapkan oleh STS."
- **Status Banner** — switch by status with proper variant + copy (tidak berubah dari v2.0).
- **Section "Detail Nominasi" (v2.2 — selalu tampil, FR-EI-13):**
  ```tsx
  {data.nomination_data && (
    <SectionCard title="Detail Nominasi">
      <dl className="grid grid-cols-2 gap-x-8 gap-y-3">
        {[
          ["Kapal",       data.nomination_data.vessel_name],
          ["ETA",         formatDateTime(data.nomination_data.eta)],
          ["Tipe Kapal",  data.nomination_data.vessel_type],
          ["Agen",        data.nomination_data.agent_name],
          ["Jenis Cargo", data.nomination_data.cargo_type],
          ["Charterer",   data.nomination_data.charterer],
          ["Towage Plan", data.nomination_data.towage_plan],
          ["Dibuat",      formatDateTime(data.nomination_data.created_at)],
          [null, null],  // spacer — grid auto-flow keeps alignment
          ["Diperbarui",  formatDateTime(data.nomination_data.updated_at)],
        ].map(([label, value], i) => label ? (
          <div key={i}>
            <dt className="text-xs font-medium text-slate-500 uppercase tracking-wide">{label}</dt>
            <dd className="text-sm text-slate-900 mt-0.5">{value ?? "—"}</dd>
          </div>
        ) : <div key={i} />)}
      </dl>
    </SectionCard>
  )}
  ```
  Jika `nomination_data` null → section tidak dirender (bukan section kosong).
- **Invoice Detail Card** (komponen baru, menggantikan section Detail Tagihan + Bank Tujuan lama):
  - **Block 1 — Vessel Ops Grid** (`<dl>` 2-col): render hanya jika `vessel_ops != null`. Field: Vessel, Crane, STS Slot, Mooring Team, ETA (formatted), Surveyor, Anchor, Est. Duration (`{n} jam`).
  - **Block 2 — Line Items Table**: render hanya jika `line_items.length > 0`. Header: Item Layanan / Volume / Rate / Jumlah. Body row: nilai kosong → "—" `text-slate-400`. `<tfoot>`:
    ```tsx
    <tfoot>
      <tr><td colSpan={3} className="text-right text-sm text-slate-500">Subtotal</td>
          <td className="text-right text-sm">{formatCurrency(subtotal, currency)}</td></tr>
      <tr><td colSpan={3} className="text-right text-sm text-slate-500">PPn ({Math.round(vat_rate*100)}%)</td>
          <td className="text-right text-sm">{formatCurrency(vat_amount, currency)}</td></tr>
      <tr className="border-t-2 border-[#0F2A4D]">
          <td colSpan={3} className="text-right font-semibold text-[#0F2A4D]">Total</td>
          <td className="text-right font-bold text-[#0F2A4D] text-base">{formatCurrency(total_amount, currency)}</td>
      </tr>
    </tfoot>
    ```
  - **Block 3 — Payment Instruction Box**: render hanya saat `status ∈ ['UNPAID', 'PAYMENT_REJECT']` AND `bank_info != null`.
    ```tsx
    <div className="rounded-xl bg-amber-50 border border-amber-200 p-4 mt-4">
      <div className="flex items-center gap-2 mb-3">
        <CreditCard className="h-4 w-4 text-amber-700" />
        <span className="text-sm font-semibold text-amber-900">Instruksi Pembayaran</span>
      </div>
      <div className="grid grid-cols-2 gap-x-6 gap-y-2">
        {/* Left col */}
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">Bank</dt>
          <dd className="text-sm text-slate-900">{bank_info.bank_name}</dd>
        </div>
        {/* Right col */}
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">Kode Bayar</dt>
          <dd className="font-mono font-semibold text-amber-900 text-sm">{bank_info.kode_bayar}</dd>
        </div>
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">No Rekening</dt>
          <dd className="font-mono font-bold text-slate-900 text-sm">{bank_info.account_number}</dd>
        </div>
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">Batas Pembayaran</dt>
          <dd className={`text-sm font-semibold ${countdown.tone === 'danger' ? 'text-rose-700' : countdown.tone === 'warning' ? 'text-amber-800' : 'text-slate-900'}`}>
            {formatDate(due_date)} {countdown.label}
          </dd>
        </div>
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">Atas Nama</dt>
          <dd className="font-semibold text-slate-900 text-sm">{bank_info.account_holder}</dd>
        </div>
        <div>
          <dt className="text-xs text-amber-700 uppercase tracking-wide">Total</dt>
          <dd className="font-bold text-[#0F2A4D] text-base">{formatCurrency(total_amount, currency)}</dd>
        </div>
      </div>
    </div>
    ```
  - **Fallback minimal:** jika `vessel_ops`, `line_items`, `bank_info` semua null/empty → render single Section Card sederhana dengan Total Tagihan + Currency + Due Date (compat untuk EPB legacy).
- **Riwayat Pembayaran** — nested cards per attempt (tidak berubah).
- **Inline Info Banner cross-link** — sama dengan v2.0.

### Currency formatter helper

File: `src/lib/format.ts`

```ts
export function formatCurrency(amount: number, currency: 'IDR' | 'USD'): string {
  if (currency === 'USD') {
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount);
  }
  return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(amount);
}

export function dueDateCountdown(dueDate: string): { label: string; tone: 'danger' | 'warning' | 'neutral' } {
  const days = Math.ceil((new Date(dueDate).getTime() - Date.now()) / 86400000);
  if (days < 0)  return { label: `Lewat ${Math.abs(days)} hari`, tone: 'danger' };
  if (days <= 3) return { label: `(${days} hari)`, tone: 'danger' };
  if (days <= 7) return { label: `(${days} hari)`, tone: 'warning' };
  return { label: '', tone: 'neutral' };
}
```

### Download endpoint (backend, FR-EI-12)

File: `internal/handlers/customer/epb_document.go`

```go
// GET /api/customer/epb-payments/:id/document
func GetEPBDocument(c echo.Context) error {
  // 1. Auth check: epb_payment milik customer auth.
  // 2. Fetch epb_pdf_url dari nomination_epb (JOIN epb_payments).
  // 3. Jika null → 404 dengan body { error: "pdf_unavailable" }.
  // 4. Fetch STS dengan Authorization: Bearer STS_API_KEY (server-side).
  // 5. Stream response body sebagai application/pdf dengan Content-Disposition attachment.
  // 6. Pada error STS (timeout, 5xx) → 502 dengan retry hint.
}
```

Catatan: jangan expose `epb_pdf_url` STS langsung ke browser (auth STS tidak boleh bocor + STS URL bisa berubah). Selalu lewat proxy.

### Bayar Form Component

File: `src/components/billing/BayarEPBForm.tsx`

```tsx
const [paidAmount, setPaidAmount] = useState<string>(item.total_amount.toString());
const [file, setFile] = useState<File | null>(null);
// ... bank_name, ref_number, payment_date

const { data: usdRate } = useQuery({ queryKey: ["config", "USD_IDR_RATE"] });
const minIDR = usdRate > 0 ? usdRate : 15000;

const isValid = (() => {
  const n = parseFloat(paidAmount.replace(/[.,]/g, ""));
  if (isNaN(n) || n < minIDR) return false;
  if (n > item.total_amount) return false;
  return file !== null;
})();
```

Field nominal: pakai input numeric dengan format thousand separator on blur. Helper text: `"Minimum: Rp {formatIDR(minIDR)} (≈ 1 USD). Maksimum: Rp {formatIDR(total_amount)}. Anda boleh membayar parsial; sisa akan ditagihkan via Invoice."`.

Inline validation:
- `< minIDR` → "Nominal kurang dari minimum (Rp {formatIDR(minIDR)})."
- `> total_amount` → "Nominal melebihi total tagihan."

Submit:
```ts
const fd = new FormData();
fd.append("file", file);
fd.append("paid_amount", paidAmount);
// ... optional fields
await fetch(`/api/customer/epb-payments/${id}/proof`, { method: "POST", body: fd });
// invalidate queries, toast success
```

---

## Acceptance Checklist

### Backend
- [ ] Migration `epb_payments` includes `total_amount`, `paid_amount`, `currency`, `due_date`, `shortfall_pushed` columns.
- [ ] M9 webhook handler (event APPROVED) sets `total_amount`, `currency`, `due_date` saat insert row baru.
- [ ] HMAC verification works on `POST /api/webhooks/sts/epb-payment-status`.
- [ ] Event `EPB_PAYMENT_REJECT` updates status + rejection_reason.
- [ ] Event `EPB_PAID` updates status + sets `nominations.status = PAYMENT_CONFIRMED` + jika shortfall > 0 → call M9c create handler.
- [ ] Event `EPB_SHORTFALL_DETECTED` idempotent (no-op jika `shortfall_pushed = true`).
- [ ] `POST /api/customer/epb-payments/:id/proof` validates `paid_amount >= minIDR` AND `<= total_amount`.
- [ ] Min IDR computed from `USD_IDR_RATE` config; fallback to `MIN_PAYMENT_IDR_FALLBACK`.
- [ ] Upload reject status `WAITING_PAYMENT_VERIFICATION` or `PAID` dengan 409.
- [ ] Update `epb_payments.status` + `nominations.status` dalam satu transaksi.
- [ ] Async send to STS includes `paid_amount` in payload.
- [ ] `GET /api/customer/epb-payments/summary` returns 3 counts (action_required, waiting_verification, paid).

### Frontend
- [ ] Sidebar nav "EPB & Invoice" link ke `/customer/billing` (yang redirect ke `/customer/billing/epb`).
- [ ] Top tabs EPB | Invoice tampil di setiap halaman billing.
- [ ] KPI Pills 3-up tampil di list page; klik aktifkan tab filter.
- [ ] Tabs filter: Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas dengan count.
- [ ] Status badges pakai label v3.3: Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas.
- [ ] Overdue indicator (rose + AlertTriangle) untuk due_date < now() dengan status != Lunas.
- [ ] Aksi column row UNPAID: tombol "Bayar" + link "Lihat Detail →".
- [ ] Detail page header: status badge + tombol "Download EPB PDF" (outline + Download icon).
- [ ] Detail page Invoice Detail Card: Block 1 Vessel Ops Grid (8 field, 2-col) render saat `vessel_ops != null`.
- [ ] Detail page Invoice Detail Card: Block 2 Line Items Table dengan Subtotal / PPn (label dinamis) / Total (bold + `text-[#0F2A4D]`).
- [ ] Detail page Invoice Detail Card: Block 3 Payment Instruction Box (amber tone, font-mono untuk Kode Bayar & No Rek) hanya tampil saat status UNPAID atau PAYMENT_REJECT.
- [ ] Countdown indicator Batas Pembayaran: ≤3 hari rose-700, 4–7 hari amber-800, >7 hari netral.
- [ ] Currency formatter: USD pakai `en-US` locale, IDR pakai `id-ID` locale; amount pakai `tabular-nums`.
- [ ] Fallback minimal saat `vessel_ops`/`line_items`/`bank_info` null (legacy EPB) — tidak boleh broken.
- [ ] Detail page Bayar form: input nominal default ke total_amount tapi editable, helper text min/max, inline validation, IDR/USD format sesuai currency.
- [ ] Submit Bayar → status berubah ke Menunggu Verifikasi; action card hide.
- [ ] Lunas dengan shortfall: info banner amber + link ke `/customer/billing/invoice/:id`.
- [ ] Download EPB PDF: klik buka file di tab baru / trigger download. Disabled + tooltip saat `epb_pdf_url` null.
- [ ] Riwayat Pembayaran nested cards dengan `paid_amount` per attempt + reason saat reject.
- [ ] Empty state: list kosong / kategori kosong dengan copy yang sesuai.

### Backend (v2.1 additions)
- [ ] `GET /api/customer/epb-payments/:id` mereturn invoice-style payload (JOIN dengan `nomination_epb` + `nomination_epb_line_items`).
- [ ] `GET /api/customer/epb-payments/:id/document` proxy stream PDF dari STS dengan Authorization Bearer (server-side). 404 saat `epb_pdf_url` null, 502 saat STS unreachable.
- [ ] `currency` field di response = `IDR` atau `USD` sesuai data STS.
- [ ] Min payment validation: saat `currency = USD`, threshold = 1 USD; saat `currency = IDR`, threshold = 1 × `USD_IDR_RATE`.

### Backend (v2.2 additions)
- [ ] `GET /api/customer/epb-payments/:id` melakukan JOIN 3-tabel: `epb_payments` ⋈ `nomination_epb` ⋈ `nominations`.
- [ ] Response JSON menyertakan field `nomination_data` dengan: `vessel_name`, `vessel_type`, `cargo_type`, `towage_plan`, `eta` (sebagai `nom_eta`), `agent_name`, `charterer`, `created_at` (sebagai `nom_created_at`), `updated_at` (sebagai `nom_updated_at`).
- [ ] `nomination_data` = `null` untuk EPB legacy (nullable, tidak error).

### Frontend (v2.2 additions)
- [ ] Section "Detail Nominasi" tampil di atas Invoice Detail Card (selalu, FR-EI-13).
- [ ] Section disembunyikan (bukan section kosong) jika `nomination_data = null`.
- [ ] Grid 2-column `dl`: Kapal, ETA | Tipe Kapal, Agen | Jenis Cargo, Charterer | Towage Plan, Dibuat | (spacer), Diperbarui.
- [ ] DateTime formatter `formatDateTime` menampilkan format `dd MMM yyyy HH:mm` (WIB-aware).
- [ ] Block 2 tfoot: baris Total pakai `border-t-2 border-[#0F2A4D]`, text `font-bold text-[#0F2A4D] text-base`.
- [ ] Block 3 Payment Instruction Box: 2-column grid (Bank + Kode Bayar / No Rekening + Batas / Atas Nama + Total). Total: `font-bold text-[#0F2A4D]`. No Rek + Kode Bayar: `font-mono font-bold/semibold text-amber-900`.
- [ ] Countdown Batas Pembayaran: ≤3 hari `text-rose-700 font-semibold`, 4–7 hari `text-amber-800`, >7 netral.
