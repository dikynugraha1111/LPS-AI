# M9 — Nomination Status & EPB Confirmation · UI Design

**Last updated:** 2026-05-13 (sync status labels v3.3) · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> **v3.3 sync notes:** Status labels diupdate sesuai BRD v3.3 (production sync): "UNPAID" → "Belum Dibayar", "Menunggu Verifikasi Pembayaran" → "Menunggu Verifikasi", "Pembayaran Dikonfirmasi" → "Lunas". Route ke M9b berubah dari `/customer/epb-invoice/:id` → `/customer/billing/epb/:id`.

> Wajib dibaca bersama `lps-design-system.md`.

---

## 1. Ringkasan

M9 menampilkan detail nominasi (status lifecycle, data submitted, history) dan EPB detail saat status APPROVED. Customer melihat tombol "Bayar EPB" yang mengarah ke M9b. M9 **tidak** mengelola upload proof — itu sepenuhnya di M9b.

Module scope lihat `module/nomination-status-payment/README.md`. FR source: FR-NP-01..FR-NP-08.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/nominations/:id` | Detail nominasi + EPB (jika ada) | Customer authenticated, owner |

List view (`/customer/nominations`) ada di M10.

---

## 3. Halaman: Nomination Detail

**Layout:** Surface A. Sidebar "Nominasi" aktif.

**Struktur 2-column:**

```
[Sidebar A]  |  ← Kembali ke Daftar Nominasi
             |
             |  Nominasi NOM-20260420-00001                  [Status Badge]
             |  MV Nusantara Star · ETA 18 Mei 2026
             |
             |  ┌──── Left col (2/3) ────────────────┬─ Right col (1/3) ──┐
             |  │                                    │                    │
             |  │ [Status Banner]                    │ [Section] Status   │
             |  │ Banner sesuai status saat ini.     │ Timeline:          │
             |  │                                    │  ● Draft           │
             |  │ [Section] Detail Kapal             │  ● Submitted       │
             |  │  Nama, IMO, Flag, DWT, LOA         │  ● Pending Review  │
             |  │                                    │  ○ Approved        │
             |  │ [Section] Detail Voyage            │  ○ EPB Confirm     │
             |  │  ETA, Port Origin/Discharge, AP    │                    │
             |  │  Jenis Kegiatan                    │ [Section] Actions  │
             |  │                                    │  (kontextual per   │
             |  │ [Section] Cargo                    │   status)          │
             |  │  Tipe, Tonase, Catatan             │                    │
             |  │                                    │                    │
             |  │ [Section] Additional Service       │                    │
             |  │  List service yang dipilih, atau   │                    │
             |  │  "Tidak ada layanan tambahan."     │                    │
             |  │                                    │                    │
             |  │ [Section] Dokumen Pendukung        │                    │
             |  │  List + download per item          │                    │
             |  │                                    │                    │
             |  │ [Section] EPB Detail (saat         │                    │
             |  │  status APPROVED ke atas)          │                    │
             |  │  - Nomor EPB                       │                    │
             |  │  - Total tagihan                   │                    │
             |  │  - Tanggal jatuh tempo             │                    │
             |  │  - Button [Bayar EPB →]            │                    │
             |  │    (navigate ke M9b)               │                    │
             |  └────────────────────────────────────┴────────────────────┘
```

### 3.1 Status Banner (varian per status)

Komponen Status Banner (design system §3.2). Variant per status nominasi:

| Status | Banner variant | Copy heading | Copy body |
|---|---|---|---|
| Draft | Neutral | "Nominasi belum dikirim" | "Lengkapi data dan klik Submit untuk mengirim nominasi ke STS." |
| Submitted / Pending | Info | "Menunggu Review STS Platform" | "Nominasi Anda sedang diverifikasi oleh tim STS. Estimasi 1–2 hari kerja." |
| Approved | Success | "Nominasi Disetujui" | "Silakan lakukan pembayaran EPB di bawah untuk melanjutkan." |
| Need Revision | Warning | "Perlu Revisi" | Pesan revisi dari STS + button "Edit Nominasi" → `/customer/nominations/:id/edit`. |
| Waiting Payment Verification | Pending verify | "Menunggu Verifikasi" | "Bukti pembayaran Anda sedang ditinjau di menu EPB & Invoice." + link |
| Payment Confirmed | Confirmed | "Lunas" | "EPB telah dibayar. Kapal dapat melanjutkan voyage sesuai jadwal." |

### 3.2 Status Timeline (right column)

Vertical timeline pakai dot bullet:
- Filled dot navy untuk step sudah selesai.
- Empty dot slate-300 untuk step belum tercapai.
- Active dot pulse animate (navy) untuk step current.
- Connector line antar dot: `border-l-2 border-slate-200 ml-1.5 my-1`.
- Per step: label + timestamp muted di bawah jika sudah terisi.

Steps:
1. Draft (terisi saat create)
2. Submitted (terisi saat submit)
3. Pending Review (terisi saat webhook diterima oleh STS)
4. Approved / Need Revision (terminal A — Need Revision back loop)
5. EPB Confirmation submitted (saat customer click "Bayar EPB" lalu submit di M9b)

### 3.3 Actions Card (kontextual)

Hanya tampil saat ada aksi yang bisa diambil customer:

- **Status Draft:** button primary "Lanjut Edit" + button ghost "Hapus Draft" (dengan dialog konfirmasi).
- **Status Need Revision:** button primary "Edit Nominasi" + display alasan revisi dari STS dalam blockquote.
- **Status Approved:** button primary "Bayar EPB" full-width → navigate ke `/customer/billing/epb/:epbPaymentId` (M9b).
- **Status lain:** Card tidak tampil ATAU tampilkan info-only message.

### 3.4 EPB Detail Section

Tampil saat `status >= APPROVED` dan `nomination.epb_number` ada.

```
EPB Detail
─────────────────────────────────────────
Nomor EPB           EPB-20260505-00001
Total Tagihan       Rp 1.250.000.000
Jatuh Tempo         15 Mei 2026
Status Pembayaran   [Belum Dibayar]  atau  [Menunggu Verifikasi]  atau  [Lunas]  dst
                    
[          Bayar EPB →          ]   ← primary, full-width
```

Button "Bayar EPB" disabled jika payment status sudah PAID. Tooltip: "Pembayaran sudah selesai."

---

## 4. Component Usage Summary

| Component | Pakai |
|---|---|
| Section Card | Setiap section di kedua kolom |
| Status Badge | Header page, EPB status |
| Status Banner | Top of left column, varian per status |
| Timeline (custom) | Right column |
| Action Card | Right column kontextual |
| Document list item | Section Dokumen Pendukung |
| Empty state | Additional Service kosong |
| Button primary | Bayar EPB, Edit Nominasi |

---

## 5. Edge Cases

| Trigger | UI behavior |
|---|---|
| Nominasi tidak ditemukan (404) | Empty state full page: icon + heading "Nominasi tidak ditemukan" + button "Kembali ke Daftar". |
| Status berubah saat user di halaman (webhook arrive) | Optional polling 30s atau WebSocket; saat berubah, banner update + toast "Status nominasi diperbarui." |
| Document download gagal | Toast error "Gagal mengunduh dokumen. Coba lagi." |
| EPB belum ter-generate meski status APPROVED | Section EPB Detail tampilkan empty state "Nomor EPB sedang dipersiapkan oleh STS. Mohon tunggu beberapa saat." |

---

## 6. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- M9b detail page: `implementation/design/m9b-epb-invoice-ui.md`
- Module scope: `module/nomination-status-payment/`
- Replit handoff: `implementation/replit-handoff/m9-nomination-status-payment.md`
- BRD: `document/brd/m9-nomination-status.md`
