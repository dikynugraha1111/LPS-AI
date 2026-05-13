# M9c — Invoice (Customer Portal) · UI Design

**Last updated:** 2026-05-13 (modul baru v3.3) · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> Wajib dibaca bersama [`lps-design-system.md`](lps-design-system.md). Untuk EPB (tagihan awal), lihat [`m9b-epb-invoice-ui.md`](m9b-epb-invoice-ui.md).

---

## 1. Ringkasan

M9c menyediakan tab **"Invoice"** dalam menu "EPB & Invoice" di Customer Portal. Mengelola pembayaran **Invoice** (tagihan settlement) yang berasal dari:
- **Shortfall EPB** — sisa pembayaran EPB yang dibayar parsial di M9b.
- **Additional Service** — tagihan layanan tambahan yang dipakai customer (Tank Cleaning, Bunkering, dll).

Lifecycle: **Belum Dibayar** → **Menunggu Verifikasi** → **Pembayaran Ditolak** (loop) atau **Lunas** (terminal). Tidak ada partial payment — Invoice harus dibayar penuh.

Module scope lihat [`module/invoice/README.md`](../../module/invoice/README.md). FR source: FR-IN-01..FR-IN-08.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/billing/invoice` | **Tab Invoice** — list seluruh Invoice customer dengan KPI pills + tabs filter | Customer authenticated |
| `/customer/billing/invoice/:id` | Detail Invoice + payment history + upload action (kontextual) | Customer authenticated, owner |

> Sidebar tetap memakai satu menu "EPB & Invoice" yang link ke `/customer/billing`. Sub-tab EPB vs Invoice ada di top of page (lihat M9b UI doc §3 untuk pattern tab navigation).

---

## 3. Halaman: Invoice List (`/customer/billing/invoice`)

```
Top tabs: [○ EPB]  [● Invoice]
────────────────────────────────────────────────────────────────
KPI Filter Pills (3 cols):
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ 2            │  │ 1            │  │ 3            │
  │ Perlu        │  │ Menunggu     │  │ Lunas        │
  │ Tindakan     │  │ Verifikasi   │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘
  (rose)            (amber)            (emerald)

Tabs filter (underline):
  [Semua (6)]  [Perlu Tindakan (2)]  [Menunggu Verifikasi (1)]  [Lunas (3)]

Filter bar (additional):
  Source: [Semua ▾] [Shortfall EPB] [Additional Service]

Table:
  No. Invoice         Source              Ref Nominasi    Amount         Jatuh Tempo    Status            Aksi
  INV-20260505-00001  Shortfall EPB       NOM-20260325..  Rp 5.000.000   20 Mei 2026    Belum Dibayar     [Bayar] Lihat Detail →
  INV-20260510-00002  Additional Service  NOM-20260420..  Rp 12.000.000  25 Mei 2026    Menunggu Verif.   Lihat Detail →
  INV-20260420-00003  Additional Service  NOM-20260301..  Rp 8.500.000   ⚠ Lewat 5 Mei  Pembayaran Ditolak Lihat Detail →
  INV-20260401-00004  Shortfall EPB       NOM-20260201..  Rp 2.000.000   10 Apr 2026    Lunas             Lihat Detail →
```

**Table specifics:**
- Komponen: Data Table dalam Section Card.
- Kolom: No. Invoice (mono), **Source badge**, Ref Nominasi (link), Amount (right-aligned IDR), Jatuh Tempo (Overdue Date Display), Status badge, Aksi.
- **Source badge variants:**
  - `EPB_SHORTFALL` → "Shortfall EPB" — Neutral pill (`bg-slate-100 border-slate-200 text-slate-700`)
  - `ADDITIONAL_SERVICE` → "Additional Service" — Info pill (`bg-blue-50 border-blue-200 text-blue-700`)
- Status badge mapping sama dengan M9b: Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas.
- Overdue indicator: pakai Overdue Date Display pattern.
- Action column untuk row `UNPAID`: tombol primary kecil "Bayar" + link "Lihat Detail →". Untuk row lain: hanya "Lihat Detail →".

**KPI Filter Pills:** Sama pattern dengan M9b list. Count diambil dari `GET /api/customer/invoices/summary`.

**Tabs filter:** Sama dengan M9b — Semua / Perlu Tindakan / Menunggu Verifikasi / Lunas.

**Additional filter "Source":** Select component di kanan tabs filter. Pilihan: Semua, Shortfall EPB, Additional Service. Filter client-side (atau via query param `?source=`).

**Empty state per tab:**
- "Belum ada Invoice. Invoice akan muncul jika ada sisa pembayaran EPB atau penggunaan Additional Service." (saat list kosong total)
- "Tidak ada Invoice dalam kategori ini." (saat tab tertentu kosong)

---

## 4. Halaman: Invoice Detail (`/customer/billing/invoice/:id`)

```
← Kembali ke daftar Invoice

Top tabs: [○ EPB]  [● Invoice]   (preserve navigation)
────────────────────────────────────────────────────────────────
INV-20260505-00001                                  [Belum Dibayar]
[Shortfall EPB] · Ref Nominasi NOM-20260325-00005

┌─ Left col (2/3) ──────────────────┬─ Right col (1/3) ─────────┐
│                                   │                           │
│ [Status Banner per status]        │ [Section] Status          │
│                                   │ Timeline:                 │
│ [Section] Asal Tagihan            │  ● Invoice dibuat         │
│  ─ Source-specific content        │  ● Pembayaran disubmit    │
│  (lihat §4.2)                     │  ○ Verifikasi STS         │
│                                   │  ○ Selesai                │
│ [Section] Detail Tagihan          │                           │
│  No. Invoice, Amount, Currency,   │ [Section] Aksi            │
│  Due Date, Status                 │  Kontextual per status    │
│                                   │  (lihat §4.3)             │
│ [Section] Bank Tujuan             │                           │
│  Bank, No. rekening, a.n.,        │                           │
│  copy button                      │                           │
│                                   │                           │
│ [Section] Riwayat Pembayaran      │                           │
│  Per attempt: file, status,       │                           │
│  reason (jika ditolak)            │                           │
└───────────────────────────────────┴───────────────────────────┘
```

### 4.1 Status Banner per Status

| Status | Banner variant | Heading | Body |
|---|---|---|---|
| Belum Dibayar | Neutral | "Menunggu Pembayaran" | "Silakan transfer sesuai detail bank di bawah dan unggah bukti pembayaran." |
| Menunggu Verifikasi | Pending verify | "Bukti Pembayaran Sedang Diverifikasi" | "Tim Finance STS sedang meninjau bukti pembayaran Anda. Estimasi 1×24 jam kerja." |
| Pembayaran Ditolak | Error | "Pembayaran Ditolak" | "Alasan: {reason}. Silakan revisi dan unggah ulang bukti pembayaran." |
| Lunas | Confirmed | "Invoice Lunas" | "Pembayaran telah diverifikasi pada {date}." |

### 4.2 Asal Tagihan Section (source-specific)

**Jika source = `EPB_SHORTFALL`:**

```jsx
<div className="rounded-xl border border-slate-200 bg-slate-50/40 p-5">
  <div className="text-xs uppercase tracking-wider text-slate-500 mb-2">Asal Tagihan</div>
  <div className="text-base font-semibold text-slate-900">Shortfall EPB</div>
  <p className="text-sm text-slate-600 mt-2">
    Tagihan ini muncul karena pembayaran EPB sebelumnya tidak penuh.
  </p>
  <div className="mt-3 flex items-center gap-2">
    <span className="text-sm text-slate-500">Parent EPB:</span>
    <a href="/customer/billing/epb/{parent_epb_id}" className="text-sm font-medium text-[#0F2A4D] hover:underline inline-flex items-center gap-1">
      {parent_epb_number} <ArrowRight className="h-3.5 w-3.5" />
    </a>
  </div>
</div>
```

**Jika source = `ADDITIONAL_SERVICE`:**

```jsx
<div className="rounded-xl border border-slate-200 bg-slate-50/40 p-5">
  <div className="text-xs uppercase tracking-wider text-slate-500 mb-2">Asal Tagihan</div>
  <div className="text-base font-semibold text-slate-900">Additional Service</div>
  <p className="text-sm text-slate-600 mt-2">
    Tagihan ini muncul dari layanan tambahan yang Anda gunakan selama voyage.
  </p>
  <div className="mt-3">
    <div className="text-xs uppercase tracking-wider text-slate-500 mb-2">Layanan yang Ditagih</div>
    <ul className="space-y-1.5">
      {service_keys.map(key => (
        <li key={key} className="flex items-center gap-2 text-sm text-slate-700">
          <Check className="h-4 w-4 text-emerald-600" />
          {SERVICE_LABELS[key]}
        </li>
      ))}
    </ul>
  </div>
</div>
```

`SERVICE_LABELS` mapping (sesuai specifications.md §9):

```js
const SERVICE_LABELS = {
  tank_cleaning: "Tank Cleaning",
  bunkering: "Pengisian Bahan Bakar atau Air Bersih",
  short_stay_temporary: "Short Stay Temporary",
  supply_logistic: "Supply Logistic",
  lay_up: "Lay Up",
  ship_chandler: "Ship Chandler",
  kapal_emergency: "Kapal Emergency",
};
```

### 4.3 Aksi Card (kontextual)

**Belum Dibayar (Bayar flow):**
- Section header "Bayar Invoice"
- Read-only display: "Total tagihan: Rp {amount}" (tidak editable — Invoice harus dibayar penuh)
- File upload dropzone (1 file, PDF/JPG/PNG, max 5 MB)
- Optional: Bank Name, Reference Number, Payment Date
- Button primary "Submit Bukti Pembayaran"

**Pembayaran Ditolak (Revisi Data flow):**
- Section header "Unggah Ulang Bukti Pembayaran"
- Display alasan reject dalam blockquote rose-50
- Field sama dengan Bayar flow
- Button primary "Submit Bukti Baru"

**Menunggu Verifikasi / Lunas:** Action card tidak tampil.

### 4.4 Riwayat Pembayaran

Sama pattern dengan M9b §5.3, tapi tanpa field `paid_amount` (Invoice selalu full payment):

```
┌──────────────────────────────────────────────┐
│ Attempt #1                       [Ditolak]   │
│ 10 Mei 2026, 10:15 WIB                       │
│ [📎 bukti-bayar-invoice.pdf]                 │
│ Alasan reject: "Bukti transfer tidak         │
│ terbaca."                                    │
└──────────────────────────────────────────────┘
```

---

## 5. Component Usage Summary

| Component (dari design system) | Dipakai di |
|---|---|
| Section Card | Semua section di list & detail |
| **Tabs underline (large, primary nav)** | Top tabs EPB / Invoice (shared dengan M9b) |
| **Tabs underline (filter)** | List filter (4 tabs) |
| **KPI Filter Pill** (§3.1) | List page atas tabs |
| Status Badge | Table column, detail header, source badge |
| Status Banner | Top of detail page |
| **Overdue Date Display** (§3.1) | Table jatuh tempo column |
| Data Table | List page |
| File upload dropzone | Aksi card (Bayar & Revisi Data) |
| Select (filter Source) | Filter bar di atas table |
| Timeline (custom) | Right col status timeline |
| Document item pattern | Proof file display di history |
| Empty state | List kosong / filter tidak match |

---

## 6. Edge Cases

| Trigger | UI behavior |
|---|---|
| Upload sukses | Optimistic UI: status berubah, banner update, history dapat item baru. Toast success. |
| Upload gagal (network) | Toast error + button "Coba Lagi". File tetap di dropzone. |
| Webhook create Invoice baru datang saat customer di page | Toast notification "Anda menerima Invoice baru" + opsi refresh list. |
| Invoice list kosong | Empty state full card: icon Receipt + "Belum ada Invoice. Akan muncul jika ada shortfall EPB atau Additional Service." |
| Parent EPB sudah ter-delete (mustahil — FK CASCADE) | Cross-link disable; fallback tampilkan "Parent EPB tidak tersedia." |
| Source tidak dikenali (data corrupted) | Show "Asal Tagihan: Lainnya" sebagai fallback, tidak crash. |

---

## 7. Cross-references

- Foundation: [`lps-design-system.md`](lps-design-system.md)
- Related M9b (parent EPB): [`m9b-epb-invoice-ui.md`](m9b-epb-invoice-ui.md)
- Related M8 (source Additional Service): [`m8-nomination-submission-ui.md`](m8-nomination-submission-ui.md)
- Module scope: [`module/invoice/`](../../module/invoice/)
- Replit handoff: [`m9c-invoice.md`](../replit-handoff/m9c-invoice.md)
- BRD: [`document/brd/m9c-invoice.md`](../../document/brd/m9c-invoice.md)
