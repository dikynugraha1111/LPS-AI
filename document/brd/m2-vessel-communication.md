# BRD Modul M2 — Vessel Communication Management

**Kategori:** Core Monitoring  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.2](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola seluruh komunikasi radio antara darat (LPS Station) dan kapal menggunakan VHF Coastal Radio terintegrasi dengan sistem AIS monitoring.

## Tujuan

- Menyediakan sarana komunikasi 2 arah yang andal antara LPS Station dan kapal.
- Merekam seluruh komunikasi radio untuk keperluan audit dan investigasi.
- Mengintegrasikan komunikasi VHF dengan data posisi AIS.

## Cakupan (In Scope)

- VHF Coastal Radio dengan integrasi software AIS monitoring.
- Perekaman otomatis seluruh komunikasi radio (voice recording).
- Log komunikasi dengan timestamp dan identifikasi kapal.
- Channel management: LPS Operator, VHF Operator, Security Operator.
- Notifikasi digital ke kapal terkait kondisi cuaca, insiden, dan informasi operasional.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-VC-01 | Sistem harus merekam otomatis seluruh komunikasi VHF antara LPS Station dan kapal beserta timestamp | System |
| FR-VC-02 | Sistem harus menampilkan log komunikasi yang dapat dicari berdasarkan waktu, kapal, dan operator | LPS Operator |
| FR-VC-03 | VHF Operator harus dapat memilih dan berpindah antar channel operasi sesuai kebutuhan | VHF Operator |
| FR-VC-04 | Sistem harus mendukung pengiriman notifikasi digital ke kapal terkait cuaca, insiden, dan instruksi operasional | LPS Operator |
| FR-VC-05 | Rekaman komunikasi harus dapat diakses kembali untuk keperluan investigasi dan audit | Admin |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | Full |
| VHF Operator | Full |
| Security Operator | View |
| Admin Kepelabuhan | Full |
| KSOP | — |
| Bea Cukai | — |
| Customer | — |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M2](../BRD-LPS-V3-Analysis.md) & [Section 3.4.2](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M2)*
- Modul terkait: [M1 — Vessel Monitoring](m1-vessel-monitoring.md)
