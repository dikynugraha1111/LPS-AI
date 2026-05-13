# M9b — EPB (Customer Portal): Technical Specifications

> **v3.3:** Scope module ini sekarang **EPB only**. Spesifikasi Invoice (settlement) ada di [`module/invoice/specifications.md`](../invoice/specifications.md).

## 1. Payment Status Lifecycle

```
[M9 webhook APPROVED handler — sistem buat row otomatis]
        │
        ▼
   Belum Dibayar (UNPAID) ──► customer klik "Bayar" ──► input nominal (≥ 1 USD) + upload proof + submit
                                                              │
                                                              ▼
                                              Menunggu Verifikasi (WAITING_PAYMENT_VERIFICATION) ──► STS verifies
                                                              │
                                                  ┌───────────┴────────────┐
                                                  ▼                        ▼
                                       Pembayaran Ditolak              Lunas (PAID)
                                       (PAYMENT_REJECT)              ──► nominations.status = PAYMENT_CONFIRMED
                                       (klik Revisi Data)            ──► jika paid_amount < total_amount:
                                                  │                       STS push EPB_SHORTFALL_DETECTED
                                                  ▼                       └─► M9c create Invoice (source=EPB_SHORTFALL)
                                          Upload + Re-Submit
                                                  │
                                                  └──► Menunggu Verifikasi (loop)
```

Statuses: `UNPAID`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_REJECT`, `PAID`

**Label mapping ke UI (Bahasa Indonesia v3.3):**
- `UNPAID` → "Belum Dibayar"
- `WAITING_PAYMENT_VERIFICATION` → "Menunggu Verifikasi"
- `PAYMENT_REJECT` → "Pembayaran Ditolak"
- `PAID` → "Lunas"

> Row `epb_payments` dibuat oleh M9 webhook handler (event `APPROVED`) dengan status `UNPAID`. Customer melakukan upload proof di M9b (bukan di M9). Setelah upload, `epb_payments.status` dan `nominations.status` sama-sama diset ke `WAITING_PAYMENT_VERIFICATION` dalam satu transaksi DB.

## 2. Partial Payment Logic

- Customer dapat input nominal pembayaran apapun selama `paid_amount >= MIN_PAYMENT_USD_EQUIV` (default: 1 USD ekuivalen IDR).
- Kurs USD→IDR di-cache dari STS (refresh harian, key System Config: `USD_IDR_RATE`). Jika cache stale (> 7 hari) atau tidak tersedia → fallback ke `MIN_PAYMENT_IDR_FALLBACK` (default Rp 15.000).
- Validasi nominal min terjadi **client-side** (UX) DAN **server-side** (security).
- `paid_amount` dikirim ke STS sebagai bagian payload payment proof.
- STS menentukan Lunas/Ditolak. Jika Lunas dengan `paid_amount < total_amount`:
  - STS update EPB status → PAID.
  - STS kirim webhook `EPB_SHORTFALL_DETECTED` ke LPS (separately atau as part of EPB_PAID payload).
  - LPS M9c handler create row `invoices` (source = `EPB_SHORTFALL`, amount = shortfall).

## 3. Database Schema

```sql
-- Core EPB payment table (owned by M9b; first row inserted by M9 webhook handler saat event APPROVED)
CREATE TABLE epb_payments (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id     UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_id            UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    status            VARCHAR(40) NOT NULL DEFAULT 'UNPAID',
    -- 'UNPAID' | 'WAITING_PAYMENT_VERIFICATION' | 'PAYMENT_REJECT' | 'PAID'
    total_amount      NUMERIC(18,2) NOT NULL,       -- nominal total dari STS
    paid_amount       NUMERIC(18,2),                -- nominal yang dibayar customer (NULL jika belum bayar)
    currency          VARCHAR(3) NOT NULL DEFAULT 'IDR',
    due_date          TIMESTAMPTZ,
    rejection_reason  TEXT,
    confirmed_at      TIMESTAMPTZ,
    shortfall_pushed  BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE setelah webhook EPB_SHORTFALL_DETECTED diteruskan ke M9c
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Payment proof uploads (one per attempt; multiple rows if rejected & re-uploaded)
CREATE TABLE epb_payment_proofs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epb_payment_id   UUID NOT NULL REFERENCES epb_payments(id) ON DELETE CASCADE,
    paid_amount      NUMERIC(18,2) NOT NULL,        -- snapshot nominal per attempt
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
CREATE INDEX idx_epb_payments_due_date      ON epb_payments(due_date);
CREATE INDEX idx_epb_proofs_payment_id      ON epb_payment_proofs(epb_payment_id);
```

## 4. STS Platform Webhook — Inbound (FR-EI-07)

### Endpoint: POST /api/webhooks/sts/epb-payment-status

**Auth:** HMAC-SHA256 signature in `X-STS-Signature` header (shared secret `STS_WEBHOOK_SECRET`).

**Payload:**
```json
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "event": "EPB_PAYMENT_REJECT | EPB_PAID | EPB_SHORTFALL_DETECTED",
  "timestamp": "ISO-8601",
  "data": {
    // For EPB_PAYMENT_REJECT:
    "rejection_reason": "Nominal transfer tidak sesuai dengan tagihan.",
    // For EPB_PAID:
    "confirmed_at": "ISO-8601",
    "paid_amount": 10000000.00,
    "shortfall_amount": 5000000.00,           // 0 atau positif
    // For EPB_SHORTFALL_DETECTED (boleh dikirim digabung dengan EPB_PAID atau terpisah):
    "shortfall_amount": 5000000.00
  }
}
```

**Processing per event:**

`EPB_PAYMENT_REJECT`:
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Update `status = PAYMENT_REJECT`, store `rejection_reason`.
3. Notification: "Bukti pembayaran ditolak. Alasan: {reason}"

`EPB_PAID`:
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Update `epb_payments.status = PAID`, set `confirmed_at`, set `paid_amount`.
3. Update `nominations.status = PAYMENT_CONFIRMED` (in transaction).
4. If `shortfall_amount > 0` (atau jika ada field `shortfall_amount` di payload):
   - Call M9c internal handler: `invoice.create(source='EPB_SHORTFALL', parent_epb_id, amount=shortfall_amount)`.
   - Set `epb_payments.shortfall_pushed = TRUE`.
5. Notification: "Pembayaran dikonfirmasi. {Jika shortfall: 'Tagihan Invoice telah ditambahkan untuk shortfall sebesar Rp {amount}.'}"

`EPB_SHORTFALL_DETECTED` (jika dikirim terpisah dari EPB_PAID):
1. Find `epb_payments` by `nomination_id` + `epb_number`.
2. Idempotency check: jika `shortfall_pushed = TRUE`, return 200 (no-op).
3. Call M9c internal handler: `invoice.create(source='EPB_SHORTFALL', parent_epb_id, amount=shortfall_amount)`.
4. Set `epb_payments.shortfall_pushed = TRUE`.

Return 200 OK immediately.

## 5. Upload Payment Proof (FR-EI-03, FR-EI-05, FR-EI-08, FR-EI-10)

### Endpoint: POST /api/customer/epb-payments/:epb_payment_id/proof

**Auth:** Customer JWT. Guard: `epb_payment` must belong to authenticated customer.

**Allowed statuses:** `UNPAID` (Bayar flow — pertama kali) atau `PAYMENT_REJECT` (Revisi Data flow). Satu endpoint untuk kedua flow, dibedakan oleh current status.

Multipart form fields:
- `file` — required, PDF/JPG/PNG, max 5MB
- `paid_amount` — **required**, decimal ≥ `MIN_PAYMENT_USD_EQUIV` (ekuivalen IDR, dihitung server-side)
- `bank_name` — optional string
- `reference_number` — optional string
- `payment_date` — optional date (YYYY-MM-DD)

Logic:
1. Validate file type (MIME check) and size.
2. Validate `paid_amount`:
   - Fetch `USD_IDR_RATE` dari System Config cache.
   - Compute `min_idr = 1 * usd_idr_rate` (atau `MIN_PAYMENT_IDR_FALLBACK` jika kurs tidak tersedia).
   - Reject 400 jika `paid_amount < min_idr` atau `paid_amount > total_amount`.
3. Save file to `uploads/epb-payments/:epb_payment_id/`.
4. Insert row into `epb_payment_proofs` (with `paid_amount` snapshot).
5. **Single transaction:**
   - Update `epb_payments.status = WAITING_PAYMENT_VERIFICATION`, `paid_amount = <input>`, `rejection_reason = NULL`, `updated_at = now()`.
   - Update `nominations.status = WAITING_PAYMENT_VERIFICATION`.
6. Async: send proof + paid_amount to STS Platform (Step 6).
7. Return 201 with updated payment state.

## 6. Send Payment Proof to STS Platform (FR-EI-08)

```
POST {STS_PLATFORM_BASE_URL}/api/epb/{epb_number}/payment-proof
Headers: Authorization: Bearer {STS_API_KEY}
Body:
{
  "lps_nomination_id": "uuid",
  "epb_number": "EPB-2026-0042",
  "paid_amount": 10000000.00,
  "currency": "IDR",
  "file_url": "https://...",
  "file_name": "bukti_bayar.pdf",
  "bank_name": "BCA",
  "reference_number": "TRF-123456",
  "payment_date": "2026-05-01",
  "uploaded_at": "ISO-8601"
}
```

Retry max 3× exponential backoff. On all retries fail: log error; status di LPS tetap `WAITING_PAYMENT_VERIFICATION`.

## 7. List & Detail EPB

### Endpoint: GET /api/customer/epb-payments

**Auth:** Customer JWT. Returns all EPB payments belonging to the customer.

**Query params:**
- `status` (optional): filter by status (`UNPAID`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_REJECT`, `PAID`, atau kategori `ACTION_REQUIRED` = UNPAID+PAYMENT_REJECT+Overdue)

Response:
```json
[
  {
    "id": "uuid",
    "nomination_id": "uuid",
    "nomination_number": "NOM-20260502-0001",
    "epb_number": "EPB-2026-0042",
    "total_amount": 15000000.00,
    "paid_amount": null,
    "currency": "IDR",
    "due_date": "2026-04-30T23:59:59Z",
    "is_overdue": true,                            // computed: due_date < now() AND status != PAID
    "status": "WAITING_PAYMENT_VERIFICATION",
    "rejection_reason": null,
    "confirmed_at": null,
    "latest_proof": { "file_name": "bukti_bayar.pdf", "uploaded_at": "..." } | null,
    "updated_at": "2026-05-01T08:01:00Z"
  }
]
```

### Endpoint: GET /api/customer/epb-payments/:id

Returns full detail of a single EPB payment including all proof upload attempts.

### Endpoint: GET /api/customer/epb-payments/summary

Returns aggregated counts per kategori (untuk KPI Filter Pill):

```json
{
  "action_required": 1,         // UNPAID + PAYMENT_REJECT + Overdue
  "waiting_verification": 2,
  "paid": 1
}
```

## 8. API Endpoints Summary

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/epb-payments | List all EPB for customer | Customer JWT |
| GET | /api/customer/epb-payments/summary | Aggregated counts per kategori | Customer JWT |
| GET | /api/customer/epb-payments/:id | EPB detail + proof history | Customer JWT |
| POST | /api/customer/epb-payments/:id/proof | Upload payment proof (Bayar flow: UNPAID, atau Revisi Data: PAYMENT_REJECT) | Customer JWT |
| POST | /api/webhooks/sts/epb-payment-status | Receive STS webhook (EPB_PAYMENT_REJECT, EPB_PAID, EPB_SHORTFALL_DETECTED) | HMAC |

## 9. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/billing | BillingLandingPage | Redirect ke /customer/billing/epb (default tab) |
| /customer/billing/epb | EPBListPage | Tab "EPB" — list semua EPB customer dengan KPI pills + tabs filter |
| /customer/billing/epb/:id | EPBDetailPage | Detail view + Bayar / Revisi Data actions |

> Catatan: sidebar tetap memakai satu menu "EPB & Invoice" yang link ke `/customer/billing`. Sub-tab EPB vs Invoice ada di top of page.
