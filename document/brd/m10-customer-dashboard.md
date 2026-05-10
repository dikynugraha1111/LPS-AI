# BRD Modul M10 — Customer Dashboard & Monitoring

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m10-customer-dashboard-monitoring.md`](../../implementation/replit-handoff/m10-customer-dashboard-monitoring.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.10](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang menyediakan halaman dashboard utama customer beserta fitur monitoring: ringkasan nominasi, pemantauan voyage yang sedang berjalan, dan akses informasi cuaca dan alert area STS Bunati.

## Tujuan

- Menyediakan single-view dashboard bagi customer untuk memantau seluruh aktivitas mereka di LPS Portal.
- Menyediakan akses cuaca dan alert untuk mendukung perencanaan operasional customer.

## Cakupan (In Scope)

- **Customer Portal Dashboard:** ringkasan nominasi aktif beserta status terkini, daftar voyage yang sedang berjalan (view-only, data dari STS Platform), dan widget cuaca real-time area STS Bunati.
- **Nomination Status page:** daftar seluruh nominasi customer beserta status terkini, termasuk Draft.
- **Voyage tracking:** pemantauan voyage aktif (view-only) termasuk posisi kapal dari data AIS LPS.
- **Weather & Alert:** data cuaca real-time dan history alert area perairan STS Bunati (view-only).
- **Document Master:** repositori dokumen pribadi customer — append-only (tidak bisa hapus/edit). Dokumen yang diupload di form nominasi otomatis tersimpan ke Document Master dan dapat dipilih kembali untuk nominasi berikutnya.

> Semua data di dashboard bersifat view-only. Sumber: LPS (cuaca, AIS) dan STS Platform (status nominasi, voyage). Document Master bersifat private per customer.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-CD-01 | Customer harus dapat memantau status voyage yang sedang berjalan (view-only) termasuk posisi kapal dari data AIS | Customer |
| FR-CD-02 | Sistem harus menampilkan halaman Nomination Status dengan daftar seluruh nominasi customer beserta status terkini, termasuk nominasi berstatus Draft | Customer |
| FR-CD-03 | Sistem harus menampilkan Customer Portal Dashboard setelah customer login, berisi: ringkasan nominasi aktif beserta status terkini, daftar voyage yang sedang berjalan (view-only, data dari STS Platform), dan widget cuaca real-time area STS Bunati | Customer |
| FR-CD-04 | Customer harus dapat melihat data cuaca real-time dan history alert area perairan STS Bunati melalui Customer Portal (view-only) | Customer |
| FR-CD-05 | Sistem harus menyediakan halaman Document Master sebagai repositori dokumen pribadi customer; customer hanya dapat mengupload dokumen baru — tidak dapat menghapus atau mengedit dokumen yang sudah diupload (append-only) | Customer |
| FR-CD-06 | Dokumen yang diupload di halaman Document Master harus menampilkan metadata: nama file, tanggal upload, ukuran file, tipe file; customer dapat mengunduh atau preview dokumen (read-only) | Customer |
| FR-CD-07 | Pada form pengajuan nominasi baru, customer harus dapat memilih antara dua metode upload dokumen: (1) Upload dari komputer lokal — file otomatis tersimpan ke Document Master; (2) Pilih dari Document Master yang sudah ada | Customer |
| FR-CD-08 | Document Master harus bersifat private per customer; dokumen satu customer tidak boleh dapat diakses oleh customer lain | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| Admin Kepelabuhan | View |
| Customer | Full |
| SuperAdmin | View |
| Role lainnya | — |

---

## Referensi

- BRD Utama: [Section 2.1 M10](../BRD-LPS-V3-Analysis.md) & [Section 3.4.10](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/customer-dashboard-monitoring/`](../../module/customer-dashboard-monitoring/)
- Replit handoff: [`m10-customer-dashboard-monitoring.md`](../../implementation/replit-handoff/m10-customer-dashboard-monitoring.md)
- Modul terkait: [M8 — Nomination Submission](m8-nomination-submission.md), [M9 — Nomination Status](m9-nomination-status.md), [M9b — EPB & Invoice](m9b-epb-invoice.md)
