# BRD Modul M4 — Incident & Emergency Management

**Kategori:** Keselamatan & Darurat  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.4](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola deteksi, pelaporan, dan penanganan insiden serta kondisi darurat di area perairan STS Bunati, termasuk insiden pencemaran dan tumpahan minyak.

## Tujuan

- Menyediakan sistem peringatan dini dan respons cepat terhadap insiden.
- Mengelola prosedur tanggap darurat secara digital dan terstruktur.
- Memastikan seluruh insiden terdokumentasi untuk keperluan audit dan regulasi.

## Cakupan (In Scope)

- Pelaporan insiden digital: form insiden, foto bukti, timestamp otomatis.
- Alert sistem untuk kondisi darurat: kebakaran, tumpahan, kecelakaan kapal.
- Oil Spill Response Management: aktivasi peralatan (Boom, Skimmer, Storage Tank).
- Eskalasi otomatis ke KSOP, Tim SAR, dan instansi terkait.
- Emergency response checklist digital.
- Post-incident report generation.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-IE-01 | LPS Operator harus dapat membuat laporan insiden digital dengan field: jenis insiden, lokasi (koordinat GPS), deskripsi, foto bukti, kapal terlibat | LPS Operator |
| FR-IE-02 | Sistem harus mengirimkan alert insiden secara otomatis ke seluruh pihak terkait (KSOP, Tim SAR, Admin) sesuai level insiden | System |
| FR-IE-03 | Sistem harus menyediakan checklist respons darurat digital yang harus diisi oleh LPS Operator | LPS Operator |
| FR-IE-04 | Sistem harus mengelola status aktivasi peralatan Oil Spill Response (Boom, Skimmer, Storage Tank) saat terjadi tumpahan | Security Operator |
| FR-IE-05 | Sistem harus mencatat seluruh timeline penanganan insiden dari deteksi hingga selesai beserta petugas yang menangani | System |
| FR-IE-06 | Sistem harus menghasilkan Post-Incident Report secara otomatis berdasarkan data penanganan insiden | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | Full |
| VHF Operator | — |
| Security Operator | Full |
| Admin Kepelabuhan | Full |
| KSOP | View |
| Bea Cukai | — |
| Customer | — |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M4](../BRD-LPS-V3-Analysis.md) & [Section 3.4.4](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M4)*
- Modul terkait: [M5 — Weather Monitoring](m5-weather-monitoring.md), [M11 — Monitoring Dashboard](m11-monitoring-dashboard.md)
