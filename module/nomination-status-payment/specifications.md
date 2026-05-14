# M9 — Nomination Status & EPB Confirmation: Technical Specifications

> **Scope revision (BRD v3.1):** M9 ends at the moment the customer submits the first proof of payment. The full payment lifecycle (Pending Review → Payment Reject → Paid) is out of M9 scope — it belongs to M9b (EPB & Invoice).

> **Update v3.4:** Tabel `nomination_epb` diperluas dengan field invoice-style (vessel ops, line items, ppn, bank info, kode bayar, pdf_url). M9 detail page menampilkan **preview** invoice-style EPB + button Download EPB PDF. Data ditulis ke `nomination_epb` saat webhook APPROVED diterima.

## 1. Nomination Status Lifecycle (M9 portion)

```
DRAFT ──────────────────────────────────── (M8)
  └─► SUBMITTED ──► PENDING ─────────────── (M8 → M9)
                       ├─► APPROVED ─────── (STS webhook → sistem otomatis buat epb_payments UNPAID)
                       │       └─► [customer buka M9b, klik Pay, upload proof]
                       │               └─► WAITING_PAYMENT_VERIFICATION ──► (STS verifies)
                       │                           ├─► PAYMENT_REJECT ──► (customer re-upload)
                       │                           └─► PAYMENT_CONFIRMED
                       └─► NEED_REVISION ──► (customer revises) ──► PENDING (re-submit)
```

Statuses managed **within M9** (on `nominations` table): `PENDING`, `APPROVED`, `NEED_REVISION`, `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_CONFIRMED`

> Setelah customer upload proof di M9b, `nominations.status` ikut berubah menjadi `WAITING_PAYMENT_VERIFICATION`. Status ini sejajar dengan `epb_payments.status`. Tidak ada status `EPB_CONFIRMATION_SUBMITTED`.

Statuses mirrored from **M9b** (`epb_payments` table): `UNPAID` (tidak ada padanannya di nominations — dibuat otomatis saat webhook APPROVED), `WAITING_PAYMENT_VERIFICATION`, `PAYMENT_REJECT`, `PAYMENT_CONFIRMED`

## 2. STS Platform Webhook — Inbound (FR-NP-01)

### Endpoint: POST /api/webhooks/sts/nomination-status
**Auth:** HMAC-SHA256 signature in `X-STS-Signature` header (shared secret).

**Payload (v3.4):**
```json
{
  "lps_nomination_id": "uuid",
  "sts_nomination_id": "sts-uuid",
  "event": "APPROVED | NEED_REVISION",
  "timestamp": "ISO-8601",
  "data": {
    // For APPROVED:
    "anchor_point": "AP-3",
    "etb": "2026-05-02T06:00:00Z",
    "estimated_duration_hours": 24,
    "epb": {
      "epb_number": "EPB-20260430-00008",
      "currency": "USD",                            // 'IDR' atau 'USD'
      "due_date": "2026-03-05T23:59:59Z",

      // ── Invoice-style data (v3.4) ──
      "vessel_ops": {
        "vessel_name": "MV Pacific Star",
        "crane": "Crane 2",
        "sts_slot": "STS-202603-001",
        "mooring_team": "Team Alpha",
        "eta": "2026-03-12T14:00:00Z",
        "surveyor": "PT Sucofindo",
        "anchor": "Anchor 3",
        "est_duration_hours": 24
      },
      "line_items": [
        { "label": "STS Fee",            "volume": "50,000 MT", "rate": "$2.50/ton", "amount": 125000.00 },
        { "label": "Biaya Jasa Tambahan", "volume": null,        "rate": null,        "amount": 0.00 }
      ],
      "subtotal": 125000.00,
      "vat_rate": 0.11,
      "vat_amount": 13750.00,
      "total_amount": 138750.00,                    // = subtotal + vat_amount

      "bank_info": {
        "bank_name": "BNI",
        "account_number": "1234567890",
        "account_holder": "PT Tata Bumi Khatulistiwa",
        "kode_bayar": "TBK-202603-001"
      },
      "epb_pdf_url": "https://sts.example.com/epb/EPB-20260430-00008.pdf"
    },
    // For NEED_REVISION:
    "revision_notes": "Dokumen Rencana Kerja tidak lengkap."
  }
}
```

**Backwards compatibility:** Field `vessel_ops`, `line_items`, `subtotal`, `vat_*`, `bank_info`, `epb_pdf_url` opsional dari sisi LPS — jika STS belum kirim, kolom NULL dan UI fallback ke tampilan minimal. Field `amount` lama (single total) tetap diterima sebagai alias `total_amount` saat field baru tidak hadir.

**Processing:**
1. Verify HMAC signature → reject with 401 if invalid.
2. Find nomination by `lps_nomination_id` → update status and store relevant data.
3. For `APPROVED` event: otomatis buat row `epb_payments` di M9b schema dengan status `UNPAID` (linked ke `nomination_epb`). Ini yang memunculkan EPB di menu M9b.
4. Trigger in-app + email notification to customer (FR-NP-04).
5. Return 200 OK immediately (process async if needed).

> Note: M9b handles its own inbound webhook events (`EPB_PAYMENT_REJECT`, `EPB_PAID`) on a separate endpoint.

## 3. View EPB & Schedule (FR-NP-02, FR-NP-03)

- EPB and schedule details are visible only when `status = APPROVED`.
- EPB data stored in `nomination_epb` table (see schema below).
- Display fields: EPB Number, Amount (formatted IDR), Due Date, Anchor Point, ETB, Estimated Duration.
- All fields are read-only — sourced from STS Platform.

## 4. Need Revision Flow (FR-NP-05)

- When `status = NEED_REVISION`, customer sees a banner with `revision_notes` from STS.
- Customer can edit: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Est. Barge, re-upload documents.
- On re-submit: same flow as M8 FR-NS-05 (send updated payload to STS API), include `sts_nomination_id` as reference.
- Status transitions back to `PENDING` after re-submit.

## 5. EPB Confirmation — View-Only Redirect to M9b (FR-NP-06, FR-NP-07)

### Trigger
- Ketika `nominations.status = APPROVED`, EPB section dan tombol "Bayar EPB" muncul.
- Tombol "Bayar EPB" menavigasi customer ke halaman M9b EPB & Invoice (`/customer/epb-invoice/:epb_payment_id`) untuk melakukan upload proof.

> **Catatan desain:** Upload proof terjadi di M9b, bukan di M9. M9 hanya menampilkan EPB detail dan menyediakan link ke M9b. Ketika customer upload proof di M9b, `nominations.status` ikut diupdate ke `WAITING_PAYMENT_VERIFICATION` oleh M9b backend.

### Setelah customer upload proof di M9b
- `nominations.status` berubah dari `APPROVED` → `WAITING_PAYMENT_VERIFICATION`.
- EPB Confirmation section di M9 diganti dengan info banner: "Menunggu Verifikasi Pembayaran — lihat status di menu EPB & Invoice."

## 6. Database Schema (M9 additions)

```sql
-- Extend nominations table
ALTER TABLE nominations ADD COLUMN anchor_point VARCHAR(50);
ALTER TABLE nominations ADD COLUMN etb TIMESTAMPTZ;
ALTER TABLE nominations ADD COLUMN estimated_duration_hours INTEGER;
ALTER TABLE nominations ADD COLUMN revision_notes TEXT;
ALTER TABLE nominations ADD COLUMN sts_nomination_id VARCHAR(100);

-- EPB data received from STS Platform (extended v3.4)
CREATE TABLE nomination_epb (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id      UUID NOT NULL REFERENCES nominations(id) ON DELETE CASCADE,
    epb_number         VARCHAR(50) NOT NULL UNIQUE,
    -- Kalkulasi
    subtotal           NUMERIC(20,2),                -- v3.4 (NULL untuk legacy)
    vat_rate           NUMERIC(5,4) DEFAULT 0.11,    -- v3.4
    vat_amount         NUMERIC(20,2),                -- v3.4
    total_amount       NUMERIC(20,2) NOT NULL,       -- = subtotal + vat_amount (atau legacy 'amount')
    currency           VARCHAR(10) NOT NULL DEFAULT 'IDR',
    due_date           TIMESTAMPTZ,
    -- Operasional voyage (v3.4)
    vessel_name        VARCHAR(255),
    crane              VARCHAR(50),
    sts_slot           VARCHAR(50),
    mooring_team       VARCHAR(100),
    eta                TIMESTAMPTZ,
    surveyor           VARCHAR(255),
    anchor             VARCHAR(50),
    est_duration_hours INTEGER,
    -- Bank instruction (v3.4)
    bank_name          VARCHAR(100),
    bank_account_no    VARCHAR(50),
    bank_account_holder VARCHAR(255),
    kode_bayar         VARCHAR(100),
    -- Dokumen (v3.4)
    epb_pdf_url        TEXT,
    -- Audit
    received_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Line items per EPB (v3.4)
CREATE TABLE nomination_epb_line_items (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    epb_id        UUID NOT NULL REFERENCES nomination_epb(id) ON DELETE CASCADE,
    label         VARCHAR(255) NOT NULL,       -- e.g. "STS Fee", "Biaya Jasa Tambahan"
    volume        VARCHAR(50),                  -- string display (e.g. "50,000 MT")
    rate          VARCHAR(50),                  -- string display (e.g. "$2.50/ton")
    amount        NUMERIC(20,2) NOT NULL,
    sort_order    INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_epb_line_items_epb_id ON nomination_epb_line_items(epb_id);

-- Immutable status change history (shared with M9b)
CREATE TABLE nomination_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nomination_id   UUID NOT NULL REFERENCES nominations(id),
    from_status     VARCHAR(50),
    to_status       VARCHAR(50) NOT NULL,
    triggered_by    VARCHAR(50) NOT NULL, -- 'customer' | 'sts_webhook' | 'system'
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_epb_nomination_id     ON nomination_epb(nomination_id);
CREATE INDEX idx_status_hist_nom_id    ON nomination_status_history(nomination_id);
```

> `epb_payments` table (for M9b EPB payment cycle) is defined in M9b specifications.

## 7. API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/customer/nominations/:id/status | Get current status + EPB + schedule | Customer JWT |
| POST | /api/customer/nominations/:id/revise | Submit revision data | Customer JWT |
| POST | /api/webhooks/sts/nomination-status | Receive STS status webhook (APPROVED / NEED_REVISION) | HMAC |

> Tidak ada endpoint EPB Confirmation upload di M9. Upload proof terjadi di M9b melalui `POST /api/customer/epb-payments/:id/proof`.

## 8. Frontend Routes

| Route | Component | Description |
|-------|-----------|-------------|
| /customer/nominations/:id | NominationStatusPage | Status, EPB, EPB Confirmation upload |
| /customer/nominations/:id/revise | NominationRevisionPage | Edit form for NEED_REVISION |
