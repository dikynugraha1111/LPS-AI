# BRD Modul M7 — Customer Authentication & Onboarding

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m7-customer-authentication.md`](../../implementation/replit-handoff/m7-customer-authentication.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.7](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola seluruh proses registrasi, verifikasi, dan autentikasi customer (Pemilik Kapal/Shipper) sebelum dapat mengakses fitur LPS Portal. Merupakan entry point untuk seluruh modul Customer Portal.

## Tujuan

- Menyediakan mekanisme self-registration bagi customer baru.
- Memfasilitasi validasi dan aktivasi akun oleh Admin.
- Menjamin hanya customer terverifikasi yang dapat mengakses portal.

## Cakupan (In Scope)

- Registrasi akun customer baru melalui LPS Portal (self-registration) dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note. Tipe Pelanggan di-default ke "Cargo Owner" oleh sistem secara otomatis.
- Upload 3 dokumen wajib saat registrasi: NPWP, NIP, Company Profile (masing-masing dengan Description, Issue Date, Expiry Date opsional).
- Customer Code di-generate otomatis oleh sistem setelah Admin menyetujui registrasi.
- Validasi dan review dokumen oleh Admin sebelum akun aktif. Setelah approve/reject, notifikasi dikirim ke customer.
- Login dan akses dashboard customer setelah akun terverifikasi.
- Data registrasi customer disinkronisasi ke STS Platform secara otomatis.
- **Admin dapat menambahkan customer baru langsung dari Admin Dashboard** tanpa melalui proses self-registration customer; akun langsung berstatus ACTIVE.
- Admin dapat melampirkan dokumen tambahan (custom documents) saat menambahkan customer secara manual, di luar 3 dokumen standar (NPWP, NIP, Company Profile).

## Status Akun Customer

| Status | Deskripsi |
|--------|-----------|
| PENDING_VALIDATION | Registrasi baru, menunggu review Admin |
| ACTIVE | Disetujui Admin atau dibuat langsung oleh Admin |
| REJECTED | Ditolak Admin |

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-CA-01 | Sistem harus menyediakan halaman registrasi bagi customer baru dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note; Tipe Pelanggan di-default ke "Cargo Owner" oleh sistem dan tidak ditampilkan sebagai field input di form registrasi | Customer |
| FR-CA-02 | Customer wajib mengupload 3 dokumen saat registrasi: NPWP (file), NIP (file), Company Profile (file), masing-masing dengan field tambahan Description (opsional), Issue Date (opsional), dan Expiry Date (opsional) | Customer |
| FR-CA-03 | Customer Code harus di-generate otomatis oleh sistem setelah Admin menyetujui registrasi; tidak diisi oleh customer | System |
| FR-CA-04 | Admin harus dapat mengakses halaman Customer Management untuk melihat daftar seluruh customer beserta statusnya (PENDING_VALIDATION, ACTIVE, REJECTED) | Admin |
| FR-CA-05 | Admin harus dapat melihat detail data registrasi customer termasuk seluruh dokumen yang diupload; Admin dapat mengunduh dokumen untuk direview | Admin |
| FR-CA-06 | Admin harus dapat mengaktifkan (Approve) akun customer berstatus PENDING_VALIDATION; sistem otomatis men-generate Customer Code dan mengirimkan notifikasi aktivasi ke email customer | Admin |
| FR-CA-07 | Admin harus dapat menolak (Reject) akun customer berstatus PENDING_VALIDATION; sistem mengirimkan notifikasi penolakan ke email customer | Admin |
| FR-CA-08 | Admin harus dapat menambahkan customer baru secara langsung dari Admin Dashboard dengan mengisi data customer yang sama seperti form registrasi; akun yang dibuat Admin langsung berstatus ACTIVE dan Customer Code langsung di-generate | Admin |
| FR-CA-09 | Saat Admin menambahkan customer secara manual, Admin dapat melampirkan dokumen tambahan (custom documents) selain 3 dokumen standar; setiap custom document dapat diberi nama/label bebas oleh Admin | Admin |
| FR-CA-10 | Customer yang sudah terverifikasi (status ACTIVE) harus dapat login dan mengakses dashboard customer | Customer |

---

## Role & Access

| Role | Akses |
|------|-------|
| Admin Kepelabuhan | Approve Reg. / Add Customer |
| Customer | Register & Login |
| SuperAdmin | View |
| Role lainnya | — |

---

## Referensi

- BRD Utama: [Section 2.1 M7](../BRD-LPS-V3-Analysis.md) & [Section 3.4.7](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/customer-authentication/`](../../module/customer-authentication/)
- Replit handoff: [`m7-customer-authentication.md`](../../implementation/replit-handoff/m7-customer-authentication.md) (v1.1)
- Delta v1.1: [`m7-customer-authentication-v1.1-delta.md`](../../implementation/replit-handoff/m7-customer-authentication-v1.1-delta.md)
- Modul terkait: [M8 — Nomination Submission](m8-nomination-submission.md)
