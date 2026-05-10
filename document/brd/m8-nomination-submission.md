# BRD Modul M8 — Nomination Request Submission

**Kategori:** Customer Portal  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Status Implementasi:** Replit handoff tersedia — [`m8-nomination-submission.md`](../../implementation/replit-handoff/m8-nomination-submission.md)  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.8](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola proses pengajuan nominasi kapal oleh customer melalui LPS Portal, dari pengisian form hingga pengiriman data ke STS Platform.

## Tujuan

- Menyediakan form pengajuan nominasi yang terstruktur.
- Mendukung penyimpanan Draft sebelum submit final.
- Meneruskan data nominasi ke STS Platform untuk proses approval dan operasional selanjutnya.

## Cakupan (In Scope)

- Form pengajuan nominasi dengan field: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Estimasi Jumlah Barge.
- Upload dokumen pendukung: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, Nomor PKK (input manual).
- Opsi simpan sebagai Draft atau Submit langsung.
- Generate Nomor Nominasi otomatis setelah submit.
- Pengiriman data nominasi ke STS Platform via API setelah submit.
- Pilihan upload dokumen: (1) Upload dari lokal → otomatis tersimpan ke Document Master; (2) Pilih dari Document Master yang sudah ada.

## Out of Scope

- Approval nominasi — dilakukan oleh STS Platform (bukan LPS).
- Scheduling dan resource assignment — dilakukan oleh STS Platform.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-NS-01 | Sistem harus menyediakan form pengajuan nominasi dengan field: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Estimasi Jumlah Barge | Customer |
| FR-NS-02 | Customer harus dapat mengupload dokumen pendukung nominasi: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, Nomor PKK (input manual) | Customer |
| FR-NS-03 | Customer harus dapat menyimpan nominasi sebagai Draft sebelum melakukan Submit | Customer |
| FR-NS-04 | Sistem harus mengenerate Nomor Nominasi otomatis setelah customer melakukan Submit | System |
| FR-NS-05 | Sistem harus mengirimkan data nominasi ke STS Platform via API setelah customer melakukan Submit | System |

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

- BRD Utama: [Section 2.1 M8](../BRD-LPS-V3-Analysis.md) & [Section 3.4.8](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/nomination-submission/`](../../module/nomination-submission/)
- Replit handoff: [`m8-nomination-submission.md`](../../implementation/replit-handoff/m8-nomination-submission.md)
- Modul terkait: [M7 — Customer Auth](m7-customer-authentication.md), [M9 — Nomination Status](m9-nomination-status.md), [M10 — Customer Dashboard](m10-customer-dashboard.md)
