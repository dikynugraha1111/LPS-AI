# M13 — Mother Vessel Master (Customer Portal) · UI Design

**Last updated:** 2026-05-16 (v1.0) · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> **v1.0:** UI design awal modul M13. Halaman khusus **Mother Vessel (MV)** — list + Add/Edit/Deactivate via approval request ke STS, detail + Riwayat Perubahan. **Tanpa tabs filter asset type** (Phase 1 MV only). Memakai pattern baru design system v1.4: Modal Dialog, Confirmation Dialog, Form Field Grid, Audit Timeline, Disclaimer Banner, Locked Row Action.

> Wajib dibaca bersama [`lps-design-system.md`](lps-design-system.md) (v1.4).

---

## 1. Ringkasan

M13 menyediakan menu **"Mother Vessel"** di sidebar Customer Portal. Customer mengelola data master MV miliknya: lihat list, tambah MV baru, edit, dan deactivate. Master MV otoritatif di STS Platform — LPS hanya UI proxy. Setiap aksi (Add/Edit/Deactivate) membuat **approval request** yang harus di-verifikasi STS Admin (lifecycle: `PENDING_APPROVAL` → `APPROVED`/`REJECTED`).

Module scope lihat [`module/mother-vessel-master/`](../../module/mother-vessel-master/). FR source: FR-MV-01..FR-MV-12.

**Surface:** A — Customer Portal, Bahasa Indonesia formal. Field label form pakai istilah maritim standar (Vessel Name, IMO, DWT, LOA) sesuai konvensi industri.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/mother-vessel` | List MV + search + pagination + Add button | Customer authenticated |
| `/customer/mother-vessel/:id` | Detail MV + Riwayat Perubahan | Customer authenticated, owner |

**Modal/Dialog (tanpa route):**
| Komponen | Trigger | Pattern design system |
|---|---|---|
| `AddMVDialog` | Klik "+ Add New MV" | Modal Dialog + Form Field Grid (§3.2 v1.4) |
| `EditMVDialog` | Klik icon Edit di row (status `ACTIVE`/`INACTIVE`/`REJECTED`) | Modal Dialog + Form Field Grid |
| `DeactivateMVDialog` | Klik icon Deactivate di row (status `ACTIVE`) | Confirmation Dialog (§3.2 v1.4) |
| `MVDetailDialog` | Klik icon View di row | Modal Dialog (detail variant) + Audit Timeline |

> Detail MV bisa berupa modal **atau** halaman penuh `/customer/mother-vessel/:id`. **Default: modal** (konsisten dengan flow tabel, tidak kehilangan konteks list). Halaman penuh dipakai bila customer akses via deep-link/refresh di route detail.

---

## 3. Sidebar Placement

Menu baru di Sidebar A (Customer Portal, §3.3 design system). Ditempatkan dalam grup **"MASTER DATA"** (grup baru bila belum ada, di bawah grup operasional Nominasi/Billing):

```
[Sidebar A]
  Dashboard
  Nominasi
  EPB & Invoice
  Document Master
  Cuaca & Alert
  ──────────────
  MASTER DATA          ← group label (text-xs uppercase tracking-wider text-slate-400, mx-3 mt-4 mb-1)
  🚢 Mother Vessel     ← nav item, icon Ship, link /customer/mother-vessel
  ──────────────
  Logout (sticky bottom)
```

- Icon: `Ship` (Lucide) — konsisten dengan ikon vessel di design system §2.7.
- Active state: `bg-[#1E3A66] text-white` (pattern Sidebar A). Inactive: `text-slate-300 hover:bg-[#13315A] hover:text-white`.
- Group label "MASTER DATA": `text-xs font-semibold uppercase tracking-wider text-slate-400/70 px-3 mt-5 mb-1.5` (di dalam navy sidebar pakai `text-slate-400`).

---

## 4. Halaman: MV List (`/customer/mother-vessel`)

**Layout:** Surface A — Customer Portal page layout (`<SidebarA />` + `<main className="flex-1 p-10 space-y-8">`).

```
[Sidebar A "Mother Vessel" aktif]  |  Mother Vessel                      [+ Add New MV]
                                   |  Kelola data Mother Vessel milik organisasi Anda.
                                   |  Setiap perubahan memerlukan persetujuan admin.
                                   |
                                   |  ┌─ Section Card ─────────────────────────────────┐
                                   |  │ [🔍 Cari asset code, nama, IMO, owner...]      │
                                   |  ├────────────────────────────────────────────────┤
                                   |  │  Aksi  Asset Code  Vessel Name  Tipe  Class  Capacity  Owner  Status │
                                   |  │  👁✏⏻  MV-0001     MV Karimun…  Gear  Pana…  50.000   PT Adaro [Active]│
                                   |  │  👁✏⏻  AST-MV-003  MV Nusan…    bulk  BV     82.000   PT Trans [Active]│
                                   |  │  👁✏⏻  AST-MV-001  MV Pacific…  bulk  BKI    75.000   PT Adaro [Inactive]│
                                   |  │  👁🔒🔒 (pending)  MV Baru…     bulk  GL     60.000   PT Anda  [Pending Approval]│
                                   |  │  👁✏⏻  AST-MV-009  MV Ditolak…  bulk  GL     45.000   PT Anda  [Rejected]│
                                   |  ├────────────────────────────────────────────────┤
                                   |  │  Baris: [10▾]   Menampilkan 1–5 dari 5    ‹ 1 › ⟳ │
                                   |  └────────────────────────────────────────────────┘
```

### 4.1 Page Header

- Title: `Mother Vessel` (`text-3xl font-semibold text-slate-900`).
- Subtitle: `Kelola data Mother Vessel milik organisasi Anda. Setiap perubahan memerlukan persetujuan admin.` (`text-sm text-slate-500 mt-1.5`).
- Action kanan: tombol primary `<Plus className="h-4 w-4" /> Add New MV` (Button Primary §3.1).

### 4.2 Search Bar

- Di dalam Section Card, baris atas sebelum tabel.
- Pattern: input dengan leading `Search` icon. Surface A pakai radius `rounded-lg` (bukan rounded-full top-bar Surface B).
- Placeholder: `Cari asset code, nama kapal, IMO, atau owner...`
- Debounce 300ms → fetch `GET /api/customer/mother-vessels?search={query}`.
- Clear button (X) muncul di kanan saat field tidak kosong.
- **Tidak ada tabs filter asset type dan tidak ada dropdown "All Assets"** — halaman khusus MV (FR-MV-02).

### 4.3 Data Table

Pakai **Data Table** (design system §3.2) di dalam Section Card.

| Kolom | Konten | Style |
|---|---|---|
| Aksi | 3 icon button: View, Edit, Deactivate | Icon-only button §3.1, gap-1 |
| Asset Code | `MV-0001` | `font-mono text-sm text-slate-600` |
| Vessel Name | `MV Karimun Jawa` | `text-sm font-medium text-slate-900` |
| Tipe Kapal | `bulk_carrier` → tampilkan title-case "Bulk Carrier" | `text-sm text-slate-600` |
| Classification | `BKI` | `text-sm text-slate-600` |
| Capacity | `50.000` (DWT, format ribuan titik) | `text-sm text-slate-600 tabular-nums text-right` |
| Owner | `PT Adaro Energy` | `text-sm text-slate-600` |
| Status | Status Badge | soft pill §3.1 |

**Status badge mapping** (design system §2.1 v1.4):

| Status API | Label (ID) | Variant |
|---|---|---|
| `ACTIVE` | Active | Confirmed (emerald) |
| `INACTIVE` | Inactive | Neutral (slate) |
| `PENDING_APPROVAL` | Pending Approval | Pending verify (teal) |
| `REJECTED` | Rejected | Error (rose) — hover/badge menampilkan alasan reject |

**Row behavior:**
- Hover row: `hover:bg-slate-50`.
- Klik row (di luar action icons) = buka MV Detail (sama dengan icon View).
- Row `REJECTED`: badge "Rejected" dengan `title={rejection_reason}` (tooltip native) + reason lengkap di Detail.

### 4.4 Action Icons per Row

Tiga icon button (Icon-only button §3.1, `h-9 w-9`):

| Icon | Aksi | Status enabled | Status locked/disabled |
|---|---|---|---|
| `Eye` (View) | Buka MV Detail | **Selalu enabled** (semua status) | — |
| `Pencil` (Edit) | Buka `EditMVDialog` | `ACTIVE`, `INACTIVE`, `REJECTED` | `PENDING_APPROVAL` → Locked Row Action (§3.2 v1.4) |
| `Power` (Deactivate) | Buka `DeactivateMVDialog` | `ACTIVE` saja | `INACTIVE`/`REJECTED` (lihat catatan) · `PENDING_APPROVAL` → Locked |

**Aturan icon kontekstual:**
- **Edit pada `REJECTED`:** enabled — customer perbaiki data lalu re-submit (jadi approval request baru). Tooltip: "Edit & ajukan ulang".
- **Deactivate pada `ACTIVE`:** enabled (icon `Power`, warna `text-slate-500`).
- **Deactivate pada `INACTIVE`:** ganti jadi aksi **Reactivate** — icon `Power` warna `text-emerald-600`, tooltip "Aktifkan kembali". Klik → buka `EditMVDialog` prefilled dengan field Status di-set `ACTIVE` (mekanisme: reactivate = Edit dengan status ACTIVE, konsisten "everything is approval request"). Tidak ada endpoint terpisah.
- **Deactivate pada `REJECTED`:** disabled (record belum pernah aktif; tidak ada yang di-deactivate). Tooltip: "Tidak tersedia untuk MV yang ditolak."
- **Semua aksi (kecuali View) pada `PENDING_APPROVAL`:** Locked Row Action — `disabled opacity-40 cursor-not-allowed` + tooltip "Sedang menunggu approval admin. Edit/Deactivate tersedia setelah approval selesai." (FR-MV-10).

### 4.5 Pagination

Pattern Data Table organism §3.3:
- Kiri-bawah: `Baris: [10 ▾]` (select 10/25/50) + counter `Menampilkan 1–10 dari N`.
- Kanan-bawah: prev `‹` · nomor halaman aktif · next `›` · tombol refresh `⟳` (re-fetch + invalidate cache).
- Disabled prev/next di boundary (opacity-40 cursor-not-allowed).

### 4.6 Empty & Error States

| Trigger | UI |
|---|---|
| Belum ada MV sama sekali | Empty state impactful (§3.2): icon `Ship h-12 w-12 text-slate-300`, heading "Belum ada Mother Vessel", body "Daftarkan MV pertama organisasi Anda. Perubahan memerlukan persetujuan admin.", CTA primary "+ Add New MV". |
| Search tidak match | Empty state ringan: "Tidak ada MV yang cocok dengan pencarian Anda." + link "Hapus pencarian". |
| STS API unreachable (5xx/timeout) | Banner error di atas tabel (Disclaimer Banner variant error — border/bg `rose-200/rose-50`, icon `AlertTriangle text-rose-600`): "Layanan master vessel sedang tidak tersedia. Coba lagi nanti." Tombol "+ Add New MV" disabled. Jika ada cache: tampilkan data cache + caption muted "Data terakhir dimuat: {timestamp}". |
| Loading | Skeleton rows (5 baris, shimmer) di area tabel — bukan blank/spinner penuh. |

---

## 5. Modal: Add / Edit MV

Pakai **Modal Dialog** + **Form Field Grid** (design system §3.2 v1.4). Width `max-w-2xl`.

```
┌─ Add New MV ───────────────────────────────────── [✕] ┐
│ Lengkapi detail di bawah. Aksi ini membuat permintaan │
│ persetujuan.                                          │
├───────────────────────────────────────────────────────┤
│ ⚠ Semua perubahan dikirim sebagai permintaan          │  ← Disclaimer Banner
│   persetujuan dan memerlukan review admin sebelum     │     (yellow severity)
│   diterapkan.                                         │
│                                                       │
│  IDENTITAS                                            │  ← group subheading
│  Asset Code            │  Vessel Name *               │
│  [MV-XXXX] (disabled)  │  [__________]                │
│  Dibuat otomatis…      │                              │
│  IMO Number *          │  Type *                      │
│  [__________]          │  [Select type ▾]            │
│  Call Sign *           │  Classification              │
│  [__________]          │  [Select classification ▾]  │
│                                                       │
│  DIMENSI & KAPASITAS                                  │  ← group subheading
│  Capacity (DWT) *      │  GRT *                        │
│  [e.g. 50000]          │  [e.g. 30000]                │
│  Length LOA (m) *      │  Beam (m) *                   │
│  [e.g. 189]            │  [e.g. 32]                   │
│  Max Draft (m) *       │  Status *                     │
│  [e.g. 12.5]           │  [Active ▾]                  │
│                                                       │
│  KEPEMILIKAN                                          │  ← group subheading
│  Owner                                                │
│  [PT Anda] (disabled)  Otomatis sesuai organisasi Anda│
├───────────────────────────────────────────────────────┤
│                              [Batal]  [Submit for Approval] │
└───────────────────────────────────────────────────────┘
```

### 5.1 Field Reference

| Group | Field | Required | Tipe | State |
|---|---|---|---|---|
| Identitas | Asset Code | — | text | **disabled** — placeholder `MV-XXXX`, helper "Dibuat otomatis saat disetujui." (Add) / read-only value (Edit) |
| Identitas | Vessel Name | ✅ | text | editable |
| Identitas | IMO Number | ✅ | text | editable, validasi 7 digit numerik |
| Identitas | Type | ✅ | select | editable (enum dari STS — placeholder "Select type") |
| Identitas | Call Sign | ✅ | text | editable |
| Identitas | Classification | — | select | editable (opsional — placeholder "Select classification") |
| Dimensi & Kapasitas | Capacity (DWT) | ✅ | number | editable, > 0 |
| Dimensi & Kapasitas | GRT | ✅ | number | editable, > 0 |
| Dimensi & Kapasitas | Length LOA (m) | ✅ | number | editable, > 0 |
| Dimensi & Kapasitas | Beam (m) | ✅ | number | editable, > 0 |
| Dimensi & Kapasitas | Max Draft (m) | ✅ | number | editable, > 0 |
| Dimensi & Kapasitas | Status | ✅ | select | `Active`/`Inactive`, default `Active` (Add) |
| Kepemilikan | Owner | — | text | **disabled** — prefilled organisasi customer, helper "Otomatis diisi sesuai organisasi Anda." |

Field grouping pakai subheading `text-xs uppercase tracking-wider text-slate-400 font-medium sm:col-span-2 pt-2` (Form Field Grid §3.2 v1.4 — form >8 field wajib di-grup).

### 5.2 Add vs Edit Behavior

| Aspek | Add | Edit |
|---|---|---|
| Modal title | "Add New MV" | "Edit MV — {asset_code}" |
| Asset Code | disabled, placeholder `MV-XXXX` | disabled, value existing (tidak bisa diubah) |
| Owner | disabled, prefilled org | disabled, value existing |
| Field lain | kosong (default Status=Active) | prefilled dari record |
| Submit action | `POST /api/customer/mother-vessels/approval-request` `{action:"ADD"}` | `{action:"EDIT", asset_id}` |
| Trigger sumber | tombol header | icon Edit row (`ACTIVE`/`INACTIVE`/`REJECTED`) |

**Edit dari `REJECTED` (re-submit):** modal title "Edit & Ajukan Ulang — {asset_code}". Banner tambahan info (blue, bukan error) di atas disclaimer: "MV ini sebelumnya ditolak. Alasan: {rejection_reason}. Perbaiki data lalu ajukan ulang."

### 5.3 Validation

- **Inline, on blur** (bukan on keystroke, bukan submit-only) — sesuai design system §3.2 Form Field Grid + UX guideline `inline-validation`.
- Error tampil di bawah field: `text-rose-600 text-xs mt-1`, border field → `border-rose-300`.
- Rules:
  - Required fields kosong → "{Label} wajib diisi."
  - IMO bukan 7 digit numerik → "Nomor IMO harus 7 digit angka."
  - Numeric ≤ 0 → "Nilai harus lebih dari 0."
- Submit gagal validasi → auto-focus field invalid pertama (`focus-management`).
- Submit `disabled` selama ada error aktif atau saat loading.

### 5.4 Submit Feedback

| State | UI |
|---|---|
| Submitting | Tombol "Submit for Approval" → spinner + disabled, label "Mengirim..." |
| Sukses (201) | Modal close. Toast success "Pengajuan terkirim. Menunggu approval admin." Tabel refresh — row baru/diubah muncul dengan badge **Pending Approval**. |
| Error 422 (IMO duplicate) | Modal tetap buka. Inline error di field IMO: "Nomor IMO sudah terdaftar." Input preserved. |
| Error 409 (sudah ada pending) | Modal tetap buka. Toast error "MV ini sedang menunggu approval. Tidak bisa diajukan ulang sampai approval selesai." |
| Error 5xx/timeout | Modal tetap buka. Toast error "Gagal mengirim ke server. Coba lagi." Input preserved, tombol kembali aktif. |

### 5.5 Dismiss Behavior

- Klik overlay / Esc / tombol Batal / icon ✕: jika **ada perubahan belum disubmit** → Confirmation Dialog kecil "Tutup tanpa menyimpan? Perubahan akan hilang." [Lanjut Edit] [Tutup]. Jika form pristine → langsung close.

---

## 6. Modal: Deactivate Confirmation

Pakai **Confirmation Dialog** (design system §3.2 v1.4). Width `max-w-md`.

```
┌──────────────────────────────────────────────┐
│  (⚠)  Nonaktifkan MV Pacific Glory?           │
│       Status akan diubah menjadi Inactive     │
│       setelah disetujui admin STS. MV tidak   │
│       akan muncul di pilihan nominasi.        │
│                                               │
│              [Batal]  [Submit for Approval]   │
└──────────────────────────────────────────────┘
```

- Icon bulat kiri: `bg-amber-50 + AlertTriangle text-amber-600` (state change butuh approval — **bukan** delete destruktif, jadi tidak pakai rose/Trash).
- Body copy sebut konsekuensi konkret: status → Inactive setelah approval + tidak muncul di nominasi.
- Tombol konfirmasi: "Submit for Approval" (primary navy, bukan destructive rose — karena ini bukan delete permanen).
- Default focus: tombol **Batal**.
- Disclaimer banner kuning kecil opsional di dalam dialog body bila ingin menekankan "memerlukan review admin" (boleh ringkas karena copy sudah menyebut "setelah disetujui admin").
- Submit → `POST /api/customer/mother-vessels/approval-request` `{action:"DEACTIVATE", asset_id}`. Feedback sama pola §5.4 (toast sukses → row jadi Pending Approval).

---

## 7. Modal/Page: MV Detail + Riwayat Perubahan

Pakai **Modal Dialog** (detail variant, `max-w-3xl`) atau halaman penuh bila deep-link. Body 2 bagian: **Detail MV** (read-only) + **Riwayat Perubahan** (Audit Timeline).

```
┌─ Detail MV — MV Pacific Glory ─────────────── [✕] ┐
│ Asset Code MV-0001          [Active]               │
├────────────────────────────────────────────────────┤
│  DATA KAPAL                                         │
│  Vessel Name   MV Pacific Glory                     │
│  IMO Number    9876543        Type      Bulk Carrier│
│  Call Sign     PXTK           Class     BKI         │
│  Capacity DWT  75.000         GRT       45.000      │
│  Length LOA    220 m          Beam      35 m        │
│  Max Draft     14.0 m         Owner     PT Anda     │
├────────────────────────────────────────────────────┤
│  RIWAYAT PERUBAHAN                                  │
│   ●─ Perubahan Data            [Rejected]           │
│   │  oleh Ishac D. · 10 Mei 2026, 08.00 WIB         │
│   │  Ditolak oleh STS Admin A · 10 Mei, 14.32 WIB   │
│   │  ┌ Alasan: Nomor IMO tidak valid. ───────────┐ │
│   │  └──────────────────────────────────────────┘ │
│   ●─ Pengajuan MV Baru         [Approved]           │
│      oleh Ishac D. · 01 Apr 2026, 09.00 WIB         │
│      Disetujui oleh STS Admin B · 01 Apr, 11.00 WIB │
├────────────────────────────────────────────────────┤
│  (jika status REJECTED)  [Edit & Ajukan Ulang]      │
│                                       [Tutup]       │
└────────────────────────────────────────────────────┘
```

### 7.1 Detail MV (read-only)

- Header modal: "Detail MV — {vessel_name}" + sub `Asset Code {asset_code}` + Status Badge di kanan.
- Body grup "DATA KAPAL": `<dl>` grid 2-kolom (pattern serupa Vessel Ops Grid) — semua field MV read-only. Label `text-xs uppercase tracking-wide text-slate-500`, value `text-sm text-slate-900`.
- Capacity/GRT format ribuan titik + `tabular-nums`. Dimensi dengan unit (`220 m`, `14.0 m`).

### 7.2 Riwayat Perubahan (Audit Timeline)

- Pakai **Audit Timeline** (design system §3.2 v1.4).
- Data dari `GET /api/customer/mother-vessels/:id/approval-history`.
- Urutan descending (terbaru di atas).
- Per item: jenis aksi (label ID: "Pengajuan MV Baru" / "Perubahan Data" / "Nonaktifkan"), submitter + timestamp, status badge, reviewer + timestamp resolution, **alasan reject** dalam box rose bila Rejected.
- Empty state: "Belum ada riwayat perubahan." (jarang — entry pertama selalu ada).

### 7.3 Footer Actions (kontekstual)

| Status MV current | Footer |
|---|---|
| `ACTIVE` / `INACTIVE` | [Tutup] saja |
| `PENDING_APPROVAL` | [Tutup] + caption muted "Pengajuan sedang ditinjau admin." |
| `REJECTED` | [Edit & Ajukan Ulang] (primary, buka EditMVDialog prefilled) + [Tutup]. Banner rose di atas timeline: "Pengajuan ditolak. Perbaiki data lalu ajukan ulang." |

---

## 8. Component Usage Summary

| Component (design system) | Dipakai di |
|---|---|
| Customer Portal page layout (Surface A) | Halaman list |
| Sidebar A — nav item baru "Mother Vessel" + grup "MASTER DATA" | Sidebar |
| Section Card | Wrapper search + tabel di list |
| Search field (rounded-lg, Surface A) | Search bar list |
| Data Table + pagination (organism §3.3) | List MV |
| Status Badge — variant Confirmed/Neutral/Pending verify/Error (§2.1 v1.4) | Kolom Status, header detail |
| Icon-only button (View/Edit/Deactivate) | Kolom Aksi |
| **Locked Row Action** (§3.2 v1.4) | Edit/Deactivate saat `PENDING_APPROVAL` |
| **Modal Dialog** (§3.2 v1.4) | Add/Edit/Detail |
| **Form Field Grid** (§3.2 v1.4) | Add/Edit form (2-kolom, grouping, disabled fields, inline validation) |
| **Disclaimer Banner** (§3.2 v1.4) | Top form Add/Edit; error banner list (variant error) |
| **Confirmation Dialog** (§3.2 v1.4) | Deactivate; dismiss-with-unsaved-changes |
| **Audit Timeline** (§3.2 v1.4) | Riwayat Perubahan di detail |
| Empty State (§3.2) | List kosong / search no match |
| Toast (§2.8) | Submit feedback (sukses/error) |

---

## 9. Edge Cases

| Trigger | UI behavior |
|---|---|
| `PENDING_APPROVAL` row | Badge "Pending Approval" (teal). Icon Edit & Deactivate → Locked Row Action (disabled + tooltip). Icon View tetap aktif. Tidak muncul di dropdown vessel M8. |
| `REJECTED` row | Badge "Rejected" (rose) + `title` = alasan reject (tooltip native). Icon Edit aktif (re-submit). Detail menampilkan banner rose + tombol "Edit & Ajukan Ulang". |
| Reactivate MV `INACTIVE` | Icon Deactivate berubah jadi Reactivate (warna emerald). Klik → EditMVDialog prefilled, Status di-set Active. Submit = approval request EDIT. |
| Submit saat STS unreachable | Toast error "Gagal mengirim ke server. Coba lagi." Modal tetap buka, input preserved. |
| Customer search lalu navigate ke detail lalu back | State search & pagination preserved (state-preservation). |
| Form ada perubahan, user klik overlay/Esc | Confirmation Dialog "Tutup tanpa menyimpan?" sebelum benar-benar close. |
| Banyak MV (>50) | Pagination wajib (default 10/page). Tidak load semua sekaligus. |
| Owner field | Selalu disabled — value dari JWT organisasi customer. Customer tidak bisa ubah (FR-MV: security). |
| Status approval berubah saat user di halaman | Polling/refetch saat cache TTL habis atau klik refresh ⟳. Saat berubah: badge update + (opsional) toast "Status MV {vessel_name} diperbarui." |
| Asset Code (Add) | Disabled, placeholder `MV-XXXX`. Value final muncul setelah `APPROVED` (di-generate STS). |

---

## 10. Accessibility Notes

- Semua icon-only action button wajib `aria-label` deskriptif ("Lihat detail MV {name}", "Edit MV {name}", "Nonaktifkan MV {name}"). Locked button: aria-label menyatakan status terkunci.
- Modal: focus trap, Esc close, focus pindah ke field pertama (form) / heading (detail) saat open, balik ke trigger saat close.
- Form: setiap input `<label htmlFor>` (tidak placeholder-only). Error pakai `aria-describedby` ke pesan error + `role="alert"`/aria-live untuk announce.
- Status badge: warna + label teks (tidak warna saja).
- Confirmation Dialog: default focus tombol Batal (cegah konfirmasi tak sengaja).
- Tabel: header `<th scope="col">`, sortable (jika diimplementasikan) pakai `aria-sort`.
- Kontras: semua text lulus AA (badge soft-pill sudah tervalidasi di design system §2.1).
- Toast: `aria-live="polite"`, tidak merebut focus, auto-dismiss 4s.

---

## 11. Cross-references

- Foundation: [`lps-design-system.md`](lps-design-system.md) (v1.4 — Modal Dialog, Confirmation Dialog, Form Field Grid, Audit Timeline, Disclaimer Banner, Locked Row Action)
- Module scope: [`module/mother-vessel-master/`](../../module/mother-vessel-master/)
- BRD: [`document/brd/m13-mother-vessel-master.md`](../../document/brd/m13-mother-vessel-master.md)
- Replit handoff: `implementation/replit-handoff/m13-mother-vessel-master.md` (akan dibuat)
- Modul terkait UI: [`m8-nomination-submission`](m8-nomination-submission-ui.md) bila ada — vessel dropdown harus filter MV `ACTIVE` (FR-MV-09)
