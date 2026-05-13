# M10 — Customer Dashboard & Monitoring · UI Design

**Last updated:** 2026-05-13 · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> Wajib dibaca bersama `lps-design-system.md`.

---

## 1. Ringkasan

M10 berisi dashboard utama customer setelah login, list nominasi global, voyage tracking (map), weather & alert, dan Document Master. M10 hanya membaca data — tidak ada flow ubah status.

Module scope lihat `module/customer-dashboard-monitoring/README.md`. FR source: FR-CD-01..FR-CD-08.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/dashboard` | Dashboard utama (aggregation) — DEFAULT setelah login | Customer authenticated |
| `/customer/nominations` | List seluruh nominasi (tabs Semua/Aktif/Draft/Selesai) | Customer authenticated |
| `/customer/voyages/:nominationId` | Tracking voyage detail (Leaflet map + AIS data) | Customer authenticated, owner |
| `/customer/documents` | Document Master (append-only library) | Customer authenticated |
| `/customer/weather` | Cuaca & Alert (snapshot + history) | Customer authenticated |

Routes terkait:
- `/customer/epb-invoice` → M9b
- `/customer/nominations/new` & `/:id/edit` → M8
- `/customer/nominations/:id` → M9

---

## 3. Halaman: Dashboard (`/customer/dashboard`)

**Mereferensi:** screenshot Dashboard customer portal yang diberikan user.

**Layout:** Surface A. Sidebar "Dashboard" aktif.

```
[Sidebar A]  |  Dashboard                                  [+ Buat Nominasi Baru]
             |  Selamat datang kembali. Berikut ringkasan aktivitas Anda.
             |
             |  ┌─ [Section] Nominasi Aktif ───────────────────────────┐
             |  │ Nominasi yang sedang dalam proses review atau         │
             |  │ menunggu pembayaran.                                  │
             |  │                                                       │
             |  │ Table: No. Nominasi | Kapal | ETA | Status | Aksi    │
             |  │ (filter: exclude DRAFT, PAYMENT_CONFIRMED, REJECTED)   │
             |  └───────────────────────────────────────────────────────┘
             |
             |  ┌─ [Section] Voyage Aktif ─────────────────────────────┐
             |  │ Kapal yang saat ini dalam pelayaran atau bersandar.   │
             |  │                                                       │
             |  │ Grid voyage cards (1 atau 2 kolom)                    │
             |  │ Per card: vessel name + nominasi number + status      │
             |  │ + AP + ETB + cargo + button "Lacak Posisi →"          │
             |  └───────────────────────────────────────────────────────┘
             |
             |  ┌─ [Section] Cuaca Saat Ini ────────────[WARNING badge]┐
             |  │ Kondisi terbaru di area STS Bunati.                   │
             |  │                                                       │
             |  │ 3-column metric tile: Tinggi Gelombang, Angin,        │
             |  │   Jarak Pandang                                       │
             |  │ Last updated text muted.                              │
             |  │ Button outlined "Lihat Detail Cuaca →"                │
             |  └───────────────────────────────────────────────────────┘
```

**Aturan filter:**
- "Nominasi Aktif" = status IN (Submitted, Pending, Approved, Need Revision, EPB_CONFIRMATION_SUBMITTED, Waiting Payment Verification, Payment Reject). EXCLUDE Draft, Payment Confirmed.
- "Voyage Aktif" = nomination dengan status >= Payment Confirmed dan vessel sedang dilacak AIS-nya.
- Cuaca: ambil snapshot terbaru dari `/api/weather/current`.

**KPI varian:** Optional jika data padat — bisa tambahkan KPI row di paling atas (jumlah nominasi aktif, jumlah EPB unpaid, dll). Tidak wajib di v1 — sesuai screenshot.

---

## 4. Halaman: Nominasi List (`/customer/nominations`)

**Mereferensi:** screenshot Nominasi yang diberikan user.

**Layout:**

```
Page header:
  Nominasi
  Daftar seluruh nominasi yang pernah Anda ajukan.        [+ Buat Nominasi Baru]

Section Card:
  Tabs:  [Semua (7)]  [Aktif]  [Draft]  [Selesai]
  ────────────────────────────────────────────────
  
  Table: No. Nominasi | Kapal | ETA | Status | Dibuat | Aksi
```

**Table behavior:**
- "Semua" = semua status, ordered by created_at desc.
- "Aktif" = exclude Draft & terminal states (Payment Confirmed, Rejected).
- "Draft" = hanya status Draft.
- "Selesai" = status Payment Confirmed (atau final state lain).
- Draft row: No. Nominasi tampil italic muted "Draft", action link "Lanjut Edit →" bukan "Lihat Detail →".
- Status badge per row sesuai mapping.

**Empty state per tab:**
- "Belum ada nominasi. Klik [+ Buat Nominasi Baru] untuk memulai." (saat semua kosong)
- "Tidak ada nominasi aktif saat ini." (saat tab Aktif kosong tapi ada di tab lain)

---

## 5. Halaman: Voyage Tracking (`/customer/voyages/:nominationId`)

**Layout:** Surface A.

```
Page header:
  Lacak Posisi — MV Sumatra Trader
  Nominasi NOM-20260325-00005

  ┌─ [Section Card with map] ────────────────────┐
  │  Leaflet map h-[640px]                       │
  │  - Vessel marker (segitiga arah)             │
  │  - Anchor point markers                      │
  │  - Bunati concession zone polygon (dashed)   │
  │  - Legend overlay kiri-bawah                 │
  └──────────────────────────────────────────────┘

  ┌─ Grid 2-col ──────────────────────────────────┐
  │ [Section] Info Voyage    │ [Section] AIS Data │
  │   ETB, ETA, AP, Status   │   MMSI, Speed,     │
  │                          │   Course, Last seen│
  └──────────────────────────────────────────────┘
```

**Map specifics:**
- Library: `react-leaflet`.
- Vessel marker: custom div icon — Lucide `Navigation` rotated by heading. Color emerald jika registered (kapal customer).
- Anchor points dari STS API: icon `Anchor` warna emerald (available) / rose (occupied).
- Bunati zone: polygon dashed border slate-400.
- Polling AIS data tiap 30s.
- Saat data AIS tidak tersedia: empty state "Data AIS belum tersedia. Posisi kapal akan ditampilkan setelah AIS aktif."

---

## 6. Halaman: Document Master (`/customer/documents`)

**Mereferensi:** screenshot Document Master yang diberikan user.

```
Page header:
  Document Master
  Perpustakaan dokumen pribadi Anda. Unggah satu kali, gunakan di banyak nominasi.

  ┌─ [Section] Unggah Dokumen Baru ──────────────────┐
  │ Dokumen yang diunggah bersifat permanen dan      │
  │ tidak dapat dihapus atau diubah.                 │
  │                                                  │
  │ [File upload dropzone — multi-file]              │
  │   "Seret & lepas file di sini, atau klik         │
  │    untuk memilih"                                │
  │   PDF, JPG, PNG — maks. 10 MB per file           │
  │   [⬆ Pilih File] (outlined button)               │
  └──────────────────────────────────────────────────┘

  ┌─ [Section] Dokumen Saya             [Search...] ─┐
  │  6 dokumen tersimpan                              │
  │ ──────────────────────────────────────────────    │
  │ List items (icon + name + meta + download):       │
  │   mv-1.jpg          JPG · 11 Mei 2026, 22.07  [⬇] │
  │   Cargo Manifest    PDF · 03 Mei 2026, 22.04  [⬇] │
  │   ...                                             │
  └──────────────────────────────────────────────────┘
```

**Append-only behavior:**
- Tidak ada tombol delete/edit/rename.
- API DELETE selalu return 403 (sudah diatur di backend).
- Pesan immutability ditegaskan di copy section header.

**Search:** filter client-side by filename (case-insensitive contains).

---

## 7. Halaman: Cuaca & Alert (`/customer/weather`)

**Mereferensi:** screenshot Cuaca & Alert yang diberikan user.

```
Page header (with icon):
  [☁ icon container] Cuaca & Alert
                     Pantau kondisi cuaca terkini dan riwayat peringatan
                     di area STS Bunati.

  ┌─ [Section] Kondisi Saat Ini ──────────────[WARNING]┐
  │ Snapshot cuaca terbaru di area anchorage.          │
  │                                                    │
  │ 3-column metric tile (uppercase labels):           │
  │   TINGGI GELOMBANG       ANGIN        JARAK PANDANG│
  │   3.0 m                  17.8 knot    4.7 km       │
  │                          251°                      │
  │                                                    │
  │ Terakhir diperbarui: 11 Mei 2026 pukul 23.46 WIB   │
  └────────────────────────────────────────────────────┘

  ┌─ [Section] Riwayat Alert ──────────────────────────┐
  │ Seluruh peringatan cuaca yang pernah dikeluarkan.  │
  │                                                    │
  │ Empty state (jika kosong):                         │
  │   "Tidak ada riwayat peringatan cuaca."            │
  │                                                    │
  │ Atau list alert items:                             │
  │   [Alert item dengan severity badge + timestamp]   │
  └────────────────────────────────────────────────────┘
```

**Page header dengan icon:** unique pattern di M10 weather. Icon container `h-10 w-10 rounded-lg bg-slate-100 flex items-center justify-center` + Lucide `CloudSun` text-slate-500. Title + subtitle di sebelah kanan icon.

**WARNING badge:** Severity Warning variant (uppercase bold). Tampil saat ada threshold breach.

**Alert item pattern:** lihat design system §3.2 Alert List Item.

---

## 8. Component Usage Summary

| Component | Halaman |
|---|---|
| Section Card | Semua halaman |
| KPI / Metric tile | Dashboard weather, Cuaca page |
| Voyage card | Dashboard |
| Status badge | Nominasi list, Dashboard |
| Tabs underline | Nominasi list |
| Table | Nominasi list, Dashboard nominasi aktif |
| File upload dropzone | Document Master |
| Document list item | Document Master |
| Map widget | Voyage tracking |
| Alert list item | Cuaca & Alert |
| Empty state | Multiple |
| Severity badge | Dashboard weather, Cuaca page |

---

## 9. Edge Cases

| Trigger | UI behavior |
|---|---|
| Customer baru (no data) | Dashboard tampilkan empty state friendly + CTA "Buat Nominasi Pertama Anda". |
| AIS belum aktif | Voyage map tampilkan area STS + pesan "Data AIS akan muncul setelah kapal masuk area pemantauan." |
| Weather API down | Section Cuaca empty state "Data cuaca tidak tersedia saat ini. Coba lagi nanti." |
| Document upload sukses | Optimistic add ke list. Toast success. |
| Document search 0 hasil | List section tampilkan "Tidak ada dokumen yang cocok dengan pencarian '{query}'." |

---

## 10. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- Module scope: `module/customer-dashboard-monitoring/`
- Replit handoff: `implementation/replit-handoff/m10-customer-dashboard-monitoring.md`
- BRD: `document/brd/m10-customer-dashboard.md`
- Prototype reference: `implementation/replit-handoff/m7-m10-customer-portal-ui-prototype.md`
