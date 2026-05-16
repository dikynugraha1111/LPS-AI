# M8 — Nomination Request Submission · UI Design

**Last updated:** 2026-05-16 (v1.1 — vessel selector terintegrasi M13) · **Surface:** A (Customer Portal) · **Status:** ACTIVE

> **v1.1:** Vessel selector tidak lagi pakai `GET /api/sts/vessels` generik. Sekarang load MV milik customer ber-status `ACTIVE` dari `GET /api/customer/mother-vessels/active` (modul M13). Tambah empty state link ke menu Mother Vessel. Lihat §4 Komponen + §5 Edge Cases.

> Wajib dibaca bersama `lps-design-system.md`.

---

## 1. Ringkasan

M8 mengelola form pembuatan nominasi baru, draft, dan submit ke STS Platform. Mencakup pilihan dokumen (upload baru atau dari Document Master), Additional Service (opsional multi-select 7 pilihan), dan submit flow.

Module scope lihat `module/nomination-submission/README.md`. FR source: FR-NS-01..FR-NS-06.

---

## 2. Page Inventory

| Route | Halaman | Akses |
|---|---|---|
| `/customer/nominations/new` | Form nominasi baru (multi-section) | Customer authenticated |
| `/customer/nominations/:id/edit` | Edit draft nominasi (same form, prefilled) | Customer authenticated, draft owner |

Halaman list nominasi (`/customer/nominations`) ada di M10.

---

## 3. Halaman: Nomination Form

**Layout:** Surface A. Sidebar customer portal aktif di "Nominasi". Main canvas: form sections stacked.

**Struktur:**

```
[Sidebar A]  |  Page header:
             |    Buat Nominasi Baru
             |    Lengkapi data kapal, cargo, dan dokumen pendukung.
             |    [← Kembali ke daftar]
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ 1. Informasi Kapal                             │
             |  │ ──────────────────────────────────────────────  │
             |  │ Mother Vessel *   [Pilih MV aktif Anda... ▾]  │
             |  │ IMO No (read-only setelah pilih MV)            │
             |  │ Flag             [____________]                │
             |  │ DWT              [____________]                │
             |  │ LOA              [____________]                │
             |  └────────────────────────────────────────────────┘
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ 2. Detail Voyage                               │
             |  │ ──────────────────────────────────────────────  │
             |  │ ETA (Tgl & Jam) *  [DD/MM/YYYY HH:MM]          │
             |  │ Port of Origin *   [____________]              │
             |  │ Port of Discharge *[____________]              │
             |  │ Anchor Point      [AP-01 ▾]                    │
             |  │ Jenis Kegiatan *  [○ STS  ○ Anchorage]         │
             |  └────────────────────────────────────────────────┘
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ 3. Cargo                                        │
             |  │ ──────────────────────────────────────────────  │
             |  │ Tipe Cargo *      [Batubara ▾]                 │
             |  │ Tonase *          [____________ MT]            │
             |  │ Catatan Cargo     [____________________]       │
             |  └────────────────────────────────────────────────┘
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ 4. Additional Service (Opsional)                │
             |  │ ──────────────────────────────────────────────  │
             |  │ Pilih layanan tambahan jika diperlukan.        │
             |  │                                                 │
             |  │ □ Tank Cleaning                                 │
             |  │ □ Pengisian Bahan Bakar atau Air Bersih         │
             |  │ □ Short Stay Temporary                          │
             |  │ □ Supply Logistic                               │
             |  │ □ Lay Up                                        │
             |  │ □ Ship Chandler                                 │
             |  │ □ Kapal Emergency                               │
             |  └────────────────────────────────────────────────┘
             |
             |  ┌────────────────────────────────────────────────┐
             |  │ 5. Dokumen Pendukung                            │
             |  │ ──────────────────────────────────────────────  │
             |  │ Tabs:  [● Upload Baru]  [○ Pilih dari Document  │
             |  │                            Master]              │
             |  │                                                 │
             |  │ — Tab 1: Dropzone upload (multi-file)          │
             |  │ — Tab 2: List dokumen dari Document Master      │
             |  │   dengan checkbox + search                      │
             |  │                                                 │
             |  │ Selected files (badge list, removable)         │
             |  └────────────────────────────────────────────────┘
             |
             |  Form actions (sticky bottom optional):
             |    [Batal]  [Simpan Draft]  [Submit Nominasi]
```

**Komponen:**

- **Each section:** Section Card (lihat design system §3.2).
- **Vessel selector (v1.1 — terintegrasi M13):** Combobox shadcn yang load **MV milik customer ber-status `ACTIVE`** dari `GET /api/customer/mother-vessels/active` (bukan lagi `GET /api/sts/vessels` generik). Pilihan tampilkan: nama kapal + asset code + IMO. Setelah pilih, simpan `mv_asset_id`; field IMO/DWT auto-fill read-only dari snapshot pilihan. MV `PENDING_APPROVAL`/`INACTIVE`/`REJECTED` tidak muncul (server-side filter). **Empty state** (customer belum punya MV ACTIVE): combobox menampilkan pesan "Belum ada Mother Vessel aktif." + link aksi "Daftarkan MV di menu Mother Vessel →" yang navigate ke `/customer/mother-vessel` (M13). Customer harus punya minimal 1 MV ACTIVE sebelum bisa submit nominasi.
- **Date-time picker:** shadcn `<Calendar>` + time input. Format: `DD/MM/YYYY HH:MM` (WIB).
- **Anchor Point selector:** shadcn `<Select>`, options dari STS API.
- **Jenis Kegiatan:** Radio group horizontal, 2 opsi.
- **Cargo type:** Select dari preset (Batubara, CPO, Minyak Mentah, dll — sesuai master STS).
- **Additional Service:** 7 checkboxes vertikal pakai shadcn `<Checkbox>` + label. Disimpan di `nomination_additional_services` (M-to-M). Default semua unchecked.
- **Dokumen Pendukung — Tabs:**
  - Tab 1 "Upload Baru": File upload dropzone (design system §3.2). Multi-file. File yang diupload otomatis tersimpan ke Document Master (lihat catatan visible di copy: "Dokumen yang diunggah akan otomatis tersimpan ke Document Master Anda.").
  - Tab 2 "Pilih dari Document Master": Search input + list dokumen Document Master dengan checkbox. List item pattern sama dengan Document Master list (icon + name + meta).
- **Selected files preview:** chip list di bawah tabs, masing-masing chip: `rounded-full bg-slate-100 px-3 py-1 text-xs text-slate-700` + icon X kanan.
- **Form actions:** flex justify-end gap-3. "Simpan Draft" outlined, "Submit Nominasi" primary, "Batal" ghost.

**Validation:**
- Required fields: ditandai `*` di label, inline error rose-600 saat blank/invalid.
- Submit disable jika required field belum lengkap; Save Draft selalu enabled.

**State:**
- Mode "new": form kosong.
- Mode "edit" (draft): form prefilled, tombol "Submit Nominasi" jika data lengkap; jika tidak, hanya "Update Draft".
- Loading saat submit: button disabled + spinner + label "Mengirim..."
- Success submit: toast success + redirect ke `/customer/nominations/:id` (detail page di M9) atau `/customer/nominations` (list di M10).

---

## 4. Component Usage Summary

| Component | Pakai |
|---|---|
| Section Card | 5 sections form |
| Tabs underline | Dokumen Pendukung (Upload Baru / Document Master) |
| File upload dropzone | Tab Upload Baru |
| Document list item | Tab Document Master |
| Checkbox group | Additional Service (7 opsi) |
| Combobox (shadcn) | Vessel selector |
| Date-time picker | ETA |
| Select | Anchor Point, Tipe Cargo |
| Radio group | Jenis Kegiatan |
| Status badge | Draft indicator (jika edit) |

---

## 5. Edge Cases

| Trigger | UI behavior |
|---|---|
| MV list gagal dimuat (STS down via M13 proxy) | Vessel combobox tampilkan empty + helper text "Gagal memuat daftar Mother Vessel. Coba lagi." + retry button. |
| Customer belum punya MV `ACTIVE` | Vessel combobox tampilkan empty state: "Belum ada Mother Vessel aktif." + link "Daftarkan MV di menu Mother Vessel →" (`/customer/mother-vessel`). Submit di-block sampai ada MV ACTIVE terpilih. |
| Anchor list STS down | Select anchor tampilkan empty + helper "Gagal memuat data dari STS. Coba lagi." + retry. |
| File upload > 10 MB | Toast error per file. File tidak ditambahkan. |
| Submit gagal di STS Platform | Toast error dengan kode error STS + tombol "Coba Lagi". Nominasi tetap sebagai Draft. |
| Draft auto-save (optional fase 2) | Indicator subtle di kanan "Draft tersimpan otomatis · 14:32" muted text. |

---

## 6. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- Module scope: `module/nomination-submission/`
- Replit handoff: `implementation/replit-handoff/m8-nomination-submission.md`
- BRD: `document/brd/m8-nomination-submission.md`
