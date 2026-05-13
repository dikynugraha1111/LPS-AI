# M9b — EPB (Customer Portal) · UI Design

**Last updated:** 2026-05-13 (v3.3 split) · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> Wajib dibaca bersama [`lps-design-system.md`](lps-design-system.md). Untuk Invoice (settlement), lihat [`m9c-invoice-ui.md`](m9c-invoice-ui.md).

---

## 1. Ringkasan

M9b menyediakan tab **"EPB"** dalam menu "EPB & Invoice" di Customer Portal. Mengelola siklus pembayaran **EPB (Estimasi Perkiraan Biaya)** — tagihan awal dengan **partial payment** (min 1 USD ekuivalen IDR). List semua EPB customer, detail per EPB, upload bukti pembayaran untuk status `Belum Dibayar` (Bayar) dan `Pembayaran Ditolak` (Revisi Data).

Lifecycle: **Belum Dibayar** → **Menunggu Verifikasi** → **Pembayaran Ditolak** (loop) atau **Lunas** (terminal). Saat Lunas dengan `paid_amount < total_amount` → Invoice otomatis dibuat di M9c.

Module scope lihat [`module/epb-invoice/README.md`](../../module/epb-invoice/README.md). FR source: FR-EI-01..FR-EI-10.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/billing` | Redirect ke `/customer/billing/epb` | Customer authenticated |
| `/customer/billing/epb` | **Tab EPB** — list seluruh EPB customer dengan KPI pills + tabs filter | Customer authenticated |
| `/customer/billing/epb/:id` | Detail EPB + payment history + upload action (kontextual) | Customer authenticated, owner |

> Menu sidebar tetap satu item: "EPB & Invoice" → link ke `/customer/billing`. Sub-tab "EPB" / "Invoice" ada di top of page.

---

## 3. Halaman: Billing Landing & Tab Navigation

Saat user klik "EPB & Invoice" di sidebar, default landing adalah tab EPB. Pattern halaman:

```
[Sidebar A "EPB & Invoice" aktif]  | Page header:
                                   |   [icon Receipt] EPB & Invoice
                                   |   Kelola dan lacak status pembayaran EPB & Invoice nominasi Anda.
                                   |
                                   | Top tabs (underline, large):
                                   |   [● EPB]  [○ Invoice]
                                   |
                                   | ──────── Content per tab ──────────
```

**Top tabs implementation:** Pakai komponen Tabs underline (design system §3.2). Size lebih besar dari tabs filter biasa karena ini adalah primary navigation antar modul. Active tab: text `text-slate-900 font-semibold`, border-bottom 2px `border-[#0F2A4D]`. Inactive: `text-slate-500 hover:text-slate-900`.

URL berubah saat switch tab (`/customer/billing/epb` vs `/customer/billing/invoice`) untuk preserve bookmark/back button.

---

## 4. Halaman: EPB List (`/customer/billing/epb`)

```
Top tabs: [● EPB]  [○ Invoice]
────────────────────────────────────────────────────────────────
KPI Filter Pills (3 cols):
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ 1            │  │ 2            │  │ 1            │
  │ Perlu        │  │ Menunggu     │  │ Lunas        │
  │ Tindakan     │  │ Verifikasi   │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘
  (rose)            (amber)            (emerald)

Tabs filter (underline):
  [Semua (4)]  [Perlu Tindakan (1)]  [Menunggu Verifikasi (2)]  [Lunas (1)]

Table:
  No. EPB         Ref Nominasi    Total          Jatuh Tempo      Status              Aksi
  EPB-20260430..  NOM-20260430..  Rp 225.000.000 20 Mei 2026      Belum Dibayar       [Bayar] Lihat Detail →
  EPB-20260425..  NOM-20260405..  Rp 97.500.000  ⚠ Lewat 13 Mei   Menunggu Verifikasi Lihat Detail →
  EPB-20260428..  NOM-20260410..  Rp 185.000.000 16 Mei 2026      Menunggu Verifikasi Lihat Detail →
  EPB-20260401..  NOM-20260325..  Rp 312.000.000 ⚠ Lewat 27 Apr   Lunas               Lihat Detail →
```

**Table specifics:**
- Komponen: Data Table (design system §3.2) dalam Section Card.
- Kolom: No. EPB (mono), Ref Nominasi (link), Total (right-aligned IDR), Jatuh Tempo (Overdue Date Display pattern), Status badge, Aksi.
- Status badge mapping (v3.3):
  - `UNPAID` → "Belum Dibayar" (Neutral pill)
  - `WAITING_PAYMENT_VERIFICATION` → "Menunggu Verifikasi" (Pending verify pill)
  - `PAYMENT_REJECT` → "Pembayaran Ditolak" (Error pill)
  - `PAID` → "Lunas" (Confirmed pill)
- Overdue indicator: pakai Overdue Date Display pattern (design system §3.1) — text rose-700 + icon AlertTriangle.
- Action column untuk row `UNPAID`: tombol primary kecil "Bayar" + link "Lihat Detail →". Untuk row lain: hanya "Lihat Detail →".

**KPI Filter Pills:** Pakai KPI Filter Pill pattern (design system §3.1). Count diambil dari `GET /api/customer/epb-payments/summary`. Click pill → set active tab filter terkait (e.g., klik "Perlu Tindakan" → tabs filter aktif "Perlu Tindakan").

**Tabs filter:** Pakai Tabs underline. Filter logic:
- **Semua**: tampilkan semua status.
- **Perlu Tindakan**: `UNPAID` + `PAYMENT_REJECT` + Overdue (semua kecuali Lunas yang sudah lewat tempo).
- **Menunggu Verifikasi**: `WAITING_PAYMENT_VERIFICATION`.
- **Lunas**: `PAID`.

**Empty state per tab:**
- "Belum ada EPB. EPB akan muncul setelah nominasi Anda disetujui." (saat list kosong total)
- "Tidak ada EPB dalam kategori ini." (saat tab tertentu kosong)

---

## 5. Halaman: EPB Detail (`/customer/billing/epb/:id`)

```
← Kembali ke daftar EPB

Top tabs: [● EPB]  [○ Invoice]   (preserve navigation)
────────────────────────────────────────────────────────────────
EPB-20260430-00008                                  [Belum Dibayar]
Ref Nominasi NOM-20260430-00008  ·  MV Nusantara Star

┌─ Left col (2/3) ──────────────────┬─ Right col (1/3) ─────────┐
│                                   │                           │
│ [Status Banner per status]        │ [Section] Status          │
│                                   │ Timeline pembayaran       │
│ [Section] Detail Tagihan          │  ● Tagihan dibuat         │
│  No. EPB, Total Amount,           │  ● Pembayaran disubmit    │
│  Paid Amount (jika ada),          │  ○ Verifikasi STS          │
│  Currency, Due Date,              │  ○ Selesai                │
│  Status                           │                           │
│                                   │ [Section] Aksi            │
│ [Section] Bank Tujuan             │  Kontextual per status    │
│  Bank, No. rekening, a.n.,        │  (lihat §5.2)             │
│  copy button                      │                           │
│                                   │                           │
│ [Section] Riwayat Pembayaran      │                           │
│  Per attempt: nominal, file,      │                           │
│  status outcome, reason           │                           │
│                                   │                           │
│ [Info banner — jika applicable]   │                           │
│  "Sisa pembayaran Rp X telah      │                           │
│   ditagihkan via Invoice          │                           │
│   [Lihat Invoice →]"              │                           │
└───────────────────────────────────┴───────────────────────────┘
```

### 5.1 Status Banner per Status

| Status | Banner variant | Heading | Body |
|---|---|---|---|
| Belum Dibayar | Neutral | "Menunggu Pembayaran" | "Silakan transfer sesuai detail bank di bawah dan unggah bukti pembayaran. Anda boleh membayar parsial (minimum 1 USD ekuivalen IDR); sisa akan ditagihkan via Invoice." |
| Menunggu Verifikasi | Pending verify | "Bukti Pembayaran Sedang Diverifikasi" | "Tim Finance STS sedang meninjau bukti pembayaran Anda. Estimasi 1×24 jam kerja." |
| Pembayaran Ditolak | Error | "Pembayaran Ditolak" | "Alasan: {reason}. Silakan revisi dan unggah ulang bukti pembayaran." |
| Lunas (tanpa shortfall) | Confirmed | "EPB Lunas" | "Pembayaran telah diverifikasi pada {date}. Voyage dapat dimulai." |
| Lunas (dengan shortfall) | Confirmed | "EPB Lunas" | "Pembayaran telah diverifikasi pada {date}. Sisa pembayaran Rp {shortfall} telah ditagihkan via Invoice. [Lihat Invoice →]" |

### 5.2 Aksi Card (kontextual)

**Belum Dibayar (Bayar flow):**
- Section header "Bayar EPB"
- Read-only display: "Total tagihan: Rp {total_amount}"
- **Nominal Pembayaran** input field:
  - Numeric input dengan format IDR (auto-format thousand separator)
  - Default value = `total_amount` (full payment) tapi editable
  - Helper text: "Minimum: Rp {min_idr} (≈ 1 USD). Maksimum: Rp {total_amount}. Anda boleh membayar parsial; sisa akan ditagihkan via Invoice."
  - Validasi inline: jika `< min_idr` → "Nominal terlalu kecil." Jika `> total_amount` → "Nominal melebihi total tagihan."
- File upload dropzone (1 file, PDF/JPG/PNG, max 5 MB)
- Optional: Bank Name, Reference Number, Payment Date
- Button primary "Submit Bukti Pembayaran"
- Setelah submit: status → Menunggu Verifikasi, banner update, action card hide.

**Pembayaran Ditolak (Revisi Data flow):**
- Section header "Unggah Ulang Bukti Pembayaran"
- Display alasan reject dalam blockquote rose-50
- Field sama dengan Bayar flow (termasuk nominal — bisa adjust kalau reject terkait jumlah)
- Button primary "Submit Bukti Baru"

**Menunggu Verifikasi / Lunas:** Action card tidak tampil.

### 5.3 Riwayat Pembayaran

List vertikal nested card per attempt:

```
┌──────────────────────────────────────────────┐
│ Attempt #2                       [Ditolak]   │
│ 05 Mei 2026, 14:32 WIB                       │
│ Nominal: Rp 10.000.000                       │
│ [📎 bukti-pembayaran-2.pdf]                  │
│ Alasan reject: "Nominal transfer tidak       │
│ sesuai dengan tagihan."                      │
├──────────────────────────────────────────────┤
│ Attempt #1                       [Ditolak]   │
│ ...                                          │
└──────────────────────────────────────────────┘
```

Setiap attempt: nested card `rounded-xl border bg-slate-50/40 p-4`. Status badge attempt kanan-atas. Field `paid_amount` di-display per attempt (snapshot). Reason hanya tampil saat ditolak.

### 5.4 Cross-link ke Invoice (Lunas dengan shortfall)

Saat status `Lunas` dan `paid_amount < total_amount`, tampilkan info banner di bawah Section Detail Tagihan:

```jsx
<div className="rounded-lg border border-amber-200 bg-amber-50/60 p-4 flex items-start gap-3">
  <Info className="h-5 w-5 text-amber-700 mt-0.5" />
  <div className="flex-1">
    <div className="text-sm font-semibold text-amber-900">Tagihan Invoice Terbuka</div>
    <p className="text-sm text-amber-800 mt-1">Sisa pembayaran sebesar <strong>Rp {shortfall}</strong> telah ditagihkan via Invoice.</p>
    <a href="/customer/billing/invoice/:invoiceId" className="inline-flex items-center gap-1 text-sm font-medium text-amber-900 hover:underline mt-2">
      Lihat Invoice <ArrowRight className="h-3.5 w-3.5" />
    </a>
  </div>
</div>
```

---

## 6. Component Usage Summary

| Component (dari design system) | Dipakai di |
|---|---|
| Section Card | Semua section di list & detail |
| **Tabs underline (large, primary nav)** | Top tabs EPB / Invoice |
| **Tabs underline (filter)** | List filter (4 tabs) |
| **KPI Filter Pill** (§3.1) | List page atas tabs |
| Status Badge | Table column, detail header |
| Status Banner | Top of detail page |
| **Overdue Date Display** (§3.1) | Table jatuh tempo column |
| **Inline Info Banner** (§3.1) | Cross-link ke Invoice |
| Data Table | List page |
| File upload dropzone | Aksi card (Bayar & Revisi Data) |
| Input numeric (IDR format) | Nominal Pembayaran field |
| Timeline (custom) | Right col status timeline |
| Document item pattern | Proof file display di history |
| Empty state | List kosong / filter tidak match |

---

## 7. Edge Cases

| Trigger | UI behavior |
|---|---|
| Customer input nominal < min IDR | Inline error rose-600 di bawah field. Submit button disabled. |
| Customer input nominal > total_amount | Inline error "Nominal melebihi total tagihan." Submit disabled. |
| Upload sukses | Optimistic UI: status berubah, banner update, history dapat item baru. Toast success. |
| Upload gagal (network) | Toast error + button "Coba Lagi". File tetap di dropzone, nominal preserved. |
| Status berubah dari webhook STS | Polling 30s atau WebSocket; update banner + toast notification. |
| Kurs USD→IDR stale/unavailable | Fallback `MIN_PAYMENT_IDR_FALLBACK`. Helper text update: "Minimum: Rp {fallback}." |
| EPB Lunas dengan shortfall | Banner status update + inline info banner di Detail Tagihan + notification + link ke Invoice. |
| EPB list kosong | Empty state full card: icon Receipt + "Belum ada EPB. Akan muncul setelah nominasi Anda disetujui." |

---

## 8. Cross-references

- Foundation: [`lps-design-system.md`](lps-design-system.md)
- Related M9c (Invoice settlement): [`m9c-invoice-ui.md`](m9c-invoice-ui.md)
- Related M9 (parent EPB Confirmation): [`m9-nomination-status-payment-ui.md`](m9-nomination-status-payment-ui.md)
- Module scope: [`module/epb-invoice/`](../../module/epb-invoice/)
- Replit handoff: [`m9b-epb-invoice.md`](../replit-handoff/m9b-epb-invoice.md)
- BRD: [`document/brd/m9b-epb-invoice.md`](../../document/brd/m9b-epb-invoice.md)
