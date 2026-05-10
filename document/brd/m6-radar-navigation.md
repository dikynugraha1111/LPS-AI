# BRD Modul M6 — Radar & Navigation Surveillance

**Kategori:** Navigasi  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.6](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola sistem RADAR dan navigasi untuk pengawasan lalu lintas kapal di seluruh area cakupan LPS Bunati, terintegrasi dengan data AIS.

## Tujuan

- Memastikan tidak ada blind spot dalam pengawasan lalu lintas kapal.
- Mengintegrasikan data RADAR dengan posisi AIS untuk akurasi tinggi.
- Mendeteksi kapal yang tidak memiliki transponder AIS aktif.

## Cakupan (In Scope)

- Sistem RADAR dengan jangkauan sesuai wilayah konsesi perairan STS Bunati.
- Overlay RADAR dengan peta AIS untuk deteksi komprehensif.
- Identifikasi kapal tanpa AIS aktif.
- Integrasi data RADAR ke LPS Operator Station.
- Alert untuk kapal yang mendekati zona larangan.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-RN-01 | Sistem harus menampilkan data RADAR dengan overlay peta AIS untuk deteksi komprehensif | System |
| FR-RN-02 | Sistem harus mengidentifikasi kapal tanpa AIS aktif melalui data RADAR | System |
| FR-RN-03 | Sistem harus memberikan alert untuk kapal yang mendekati zona larangan | System |
| FR-RN-04 | Sistem harus mengintegrasikan data RADAR ke LPS Operator Station | System |

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

- BRD Utama: [Section 2.1 M6](../BRD-LPS-V3-Analysis.md) & [Section 3.4.6](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M6)*
- Modul terkait: [M1 — Vessel Monitoring](m1-vessel-monitoring.md)
