# M9b — EPB & Invoice · UI Design

**Last updated:** 2026-05-13 · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> Wajib dibaca bersama `lps-design-system.md`.

---

## 1. Ringkasan

M9b adalah menu "EPB & Invoice" di Customer Portal. Mengelola siklus pembayaran EPB: list semua EPB customer, detail per EPB, upload bukti pembayaran untuk status `UNPAID` (Pay) dan `PAYMENT_REJECT` (Revision Data). Tabel `epb_payments` punya 4 status: UNPAID → WAITING_PAYMENT_VERIFICATION → PAYMENT_REJECT (loop) atau PAID (terminal).

Module scope lihat `module/epb-invoice/README.md`. FR source: FR-EI-01..FR-EI-09.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/epb-invoice` | List semua EPB customer dengan tabs filter | Customer authenticated |
| `/customer/epb-invoice/:id` | Detail EPB + payment history + upload action (kontextual) | Customer authenticated, owner |

---

## 3. Halaman: EPB List (`/customer/epb-invoice`)

**Layout:** Surface A. Sidebar "EPB & Invoice" aktif.

**Struktur:**

```
[Sidebar A]  |  EPB & Invoice
             |  Kelola pembayaran EPB untuk seluruh nominasi Anda.
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ Tabs:  [Semua (12)] [Unpaid (3)]               │
             |  │        [Menunggu Verifikasi (2)] [Reject (1)]  │
             |  │        [Paid (6)]                              │
             |  │ ─────────────────────────────────────────────   │
             |  │                                                 │
             |  │ No. EPB         Kapal           Nominasi   ...  │
             |  │ EPB-...-00001  MV Nusantara    NOM-...    ...  │
             |  │                                                 │
             |  │ (table dengan kolom:                            │
             |  │  No EPB, Kapal, Nominasi, Total, Jatuh Tempo,   │
             |  │  Status, Aksi)                                  │
             |  └────────────────────────────────────────────────┘
```

**Tabs filter:** underline style. Counts dalam kurung muted.

**Table kolom:**

| Kolom | Format |
|---|---|
| No. EPB | `EPB-20260505-00001` monospace style |
| Kapal | Nama kapal |
| Nominasi | Link ke `/customer/nominations/:id` (link-action style) |
| Total | `Rp 1.250.000.000` right-aligned |
| Jatuh Tempo | `15 Mei 2026` |
| Status | Status badge |
| Aksi | Link-action "Lihat Detail →" |

**Status mapping di table:**
- UNPAID → Neutral uppercase
- WAITING_PAYMENT_VERIFICATION → Pending verify
- PAYMENT_REJECT → Error "Pembayaran Ditolak"
- PAID → Confirmed "Lunas"

---

## 4. Halaman: EPB Detail (`/customer/epb-invoice/:id`)

**Layout:** Surface A. Breadcrumb `EPB & Invoice > {EPB Number}`.

**Struktur 2-column:**

```
[Sidebar A]  |  ← Kembali ke daftar EPB
             |
             |  EPB-20260505-00001                          [Status Badge]
             |  MV Nusantara Star · Nominasi NOM-20260420-00001
             |
             |  ┌──── Left col (2/3) ────────────────┬─ Right col (1/3) ──┐
             |  │                                    │                    │
             |  │ [Status Banner]                    │ [Section] Status   │
             |  │                                    │ Timeline pembayaran│
             |  │ [Section] Detail Tagihan           │  ● Issued           │
             |  │  Total, jatuh tempo, breakdown     │  ● Proof Submitted │
             |  │  line items dari STS               │  ○ Verified        │
             |  │                                    │                    │
             |  │ [Section] Bank Tujuan              │ [Section] Aksi     │
             |  │  Bank name, account number,        │  Kontextual:       │
             |  │  account holder, copy button       │  - UNPAID: Bayar   │
             |  │                                    │  - REJECT: Revision│
             |  │ [Section] Riwayat Pembayaran       │  - WAITING/PAID:   │
             |  │  Per attempt: timestamp,           │    info only       │
             |  │  amount, proof file, status,       │                    │
             |  │  rejection reason (jika reject)    │                    │
             |  │                                    │                    │
             |  └────────────────────────────────────┴────────────────────┘
```

### 4.1 Status Banner per Payment Status

| Status | Banner variant | Heading | Body |
|---|---|---|---|
| UNPAID | Neutral | "Menunggu Pembayaran" | "Silakan transfer sesuai detail bank di bawah dan unggah bukti pembayaran." |
| WAITING_PAYMENT_VERIFICATION | Pending verify | "Bukti Pembayaran Sedang Diverifikasi" | "Tim Finance STS sedang meninjau bukti pembayaran Anda. Estimasi 1×24 jam kerja." |
| PAYMENT_REJECT | Error | "Pembayaran Ditolak" | "Alasan: {reason}. Silakan revisi dan unggah ulang bukti pembayaran." |
| PAID | Confirmed | "Pembayaran Selesai" | "Pembayaran telah diverifikasi pada {date}. Nominasi siap diproses." |

### 4.2 Bank Account Card

```
Bank Tujuan
──────────────────────────────────────
Bank Mandiri
1234-5678-9012-3456    [📋 Salin]
a.n. PT. TBK
```

Setiap nomor: clickable copy ke clipboard, toast confirm "Disalin ke clipboard".

### 4.3 Aksi Card (kontextual)

**UNPAID (Pay flow):**
- Section header "Unggah Bukti Pembayaran"
- File upload dropzone (1 file, PDF/JPG/PNG, max 10 MB)
- Optional field: catatan / nomor referensi transfer
- Button primary "Submit Bukti Pembayaran"
- Setelah submit: status berubah ke WAITING_PAYMENT_VERIFICATION, banner update, action card hide.

**PAYMENT_REJECT (Revision Data flow):**
- Section header "Unggah Ulang Bukti Pembayaran"
- Display alasan reject dari verifikator dalam blockquote rose-50
- Sama dengan UNPAID upload flow
- Button primary "Submit Bukti Baru"

**WAITING_PAYMENT_VERIFICATION / PAID:** Action card tidak tampil atau tampilkan info message.

### 4.4 Riwayat Pembayaran (Payment History)

List vertikal, satu item per attempt. Format:

```
┌──────────────────────────────────────────────┐
│ Attempt #2                       [REJECT]    │
│ 05 Mei 2026, 14:32 WIB                       │
│ Rp 1.250.000.000                             │
│ [📎 bukti-pembayaran-2.pdf]                  │
│ Alasan reject: "Nominal transfer tidak       │
│ sesuai dengan tagihan."                      │
├──────────────────────────────────────────────┤
│ Attempt #1                       [REJECT]    │
│ ...                                          │
└──────────────────────────────────────────────┘
```

Setiap attempt: nested card `rounded-xl border bg-slate-50/40 p-4`. Status badge kanan-atas. Reason hanya tampil saat status reject.

---

## 5. Component Usage Summary

| Component | Pakai |
|---|---|
| Section Card | Semua section di list & detail |
| Tabs underline | List filter (5 tabs) |
| Status Badge | Table column, header detail, history items |
| Status Banner | Top of detail page |
| File upload dropzone | Aksi card (Pay & Revision) |
| Timeline (custom) | Right col status timeline |
| Document item pattern | Proof file display di history |
| Empty state | List dengan filter tidak match |

---

## 6. Edge Cases

| Trigger | UI behavior |
|---|---|
| Upload sukses | Optimistic UI: status berubah, banner update, history dapat item baru. Toast success. |
| Upload gagal (network) | Toast error + button "Coba Lagi". File tetap di dropzone. |
| Upload duplikat (race condition) | Backend tolak → toast warning "Bukti sudah diunggah sebelumnya. Refresh halaman." |
| EPB list kosong | Empty state full card: icon Receipt + "Belum ada tagihan EPB." |
| Status berubah dari webhook STS (mis. PAYMENT_REJECT arrive) | Polling 30s atau WebSocket; update banner + toast notifikasi. |

---

## 7. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- Related M9: `implementation/design/m9-nomination-status-payment-ui.md`
- Module scope: `module/epb-invoice/`
- Replit handoff: `implementation/replit-handoff/m9b-epb-invoice.md`
- BRD: `document/brd/m9b-epb-invoice.md`
