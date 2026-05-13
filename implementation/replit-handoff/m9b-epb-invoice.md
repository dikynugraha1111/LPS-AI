# Replit Handoff — M9b: EPB (Customer Portal)

**Version:** 2.0 (v3.3 split — EPB only, partial payment)

> **Version history:**
> - **v1.0** — 2026-05-07: Initial M9b handoff (gabungan EPB & Invoice scope, no partial payment).
> - **v2.0** — 2026-05-13: Split M9b → EPB only. Invoice (settlement) pindah ke M9c. Tambah partial payment (min 1 USD ekuivalen IDR). Status labels disinkronkan ke Bahasa Indonesia natural. Route berubah dari `/customer/epb-invoice` → `/customer/billing/epb`. Webhook `EPB_SHORTFALL_DETECTED` ditambah untuk trigger Invoice creation di M9c.

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

1. **Design system master:** [`implementation/design/lps-design-system.md`](../design/lps-design-system.md) — v1.2 (status label sync, KPI Filter Pill, Overdue Date Display, Inline Info Banner).
2. **Per-modul UI design:** [`implementation/design/m9b-epb-invoice-ui.md`](../design/m9b-epb-invoice-ui.md) — list page dengan KPI pills + tabs filter, detail page 2-column, status banner per 4 payment status, **Bayar form dengan nominal input + validasi partial payment**, cross-link ke Invoice saat shortfall.

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

Full detail: EPB record + all proof attempts (DESC by uploaded_at).
Guard: customer_id check via JOIN to nominations.

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

## Step 9: Frontend — EPB Detail Page

File: `src/pages/customer/billing/EPBDetailPage.tsx`

Layout 2-column (lihat m9b-epb-invoice-ui.md §5).

Key sections:
- **Status Banner** — switch by status with proper variant + copy.
- **Detail Tagihan** — Total Amount, Paid Amount (jika ada), Currency, Due Date, Status. Jika Lunas dengan shortfall: tampilkan info banner amber + link ke `/customer/billing/invoice/:invoiceId` (resolved dari query).
- **Bank Tujuan** — static config (system config keys `BANK_NAME`, `BANK_ACCOUNT_NUMBER`, `BANK_ACCOUNT_HOLDER`) atau dari STS API. Tombol copy-to-clipboard.
- **Riwayat Pembayaran** — nested cards per attempt.

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
- [ ] Detail page Bayar form: input nominal default ke total_amount tapi editable, helper text min/max, inline validation, IDR thousand separator.
- [ ] Submit Bayar → status berubah ke Menunggu Verifikasi; action card hide.
- [ ] Lunas dengan shortfall: info banner amber + link ke `/customer/billing/invoice/:id`.
- [ ] Bank account card dengan copy-to-clipboard + toast.
- [ ] Riwayat Pembayaran nested cards dengan `paid_amount` per attempt + reason saat reject.
- [ ] Empty state: list kosong / kategori kosong dengan copy yang sesuai.
