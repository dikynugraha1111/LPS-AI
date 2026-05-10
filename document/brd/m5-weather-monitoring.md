# BRD Modul M5 — Weather Monitoring & Alert System

**Kategori:** Keselamatan & Darurat  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.5](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang memantau kondisi cuaca dan gelombang di area perairan STS Bunati secara real-time, serta menghasilkan peringatan otomatis jika kondisi melebihi batas aman operasi.

## Tujuan

- Memastikan keselamatan operasi STS dengan pemantauan cuaca real-time.
- Memberikan peringatan dini kepada seluruh kapal di area STS.
- Mendukung analisis produktivitas berdasarkan data cuaca historis.

## Cakupan (In Scope)

- Integrasi Weather API untuk data cuaca real-time.
- Alert otomatis jika tinggi gelombang > 2.5m (warning) dan > 3m (operasi berhenti).
- Weather forecast H+24 dan H+48 untuk perencanaan operasi.
- Weather log otomatis terintegrasi dengan data operasional STS.
- Dashboard cuaca real-time untuk LPS Operator dan kapal.

## Batasan Operasional

| Threshold | Level | Aksi |
|-----------|-------|------|
| Gelombang > 2.5m | Warning | Alert ke seluruh kapal |
| Gelombang > 3m | Critical | Rekomendasi penghentian operasi |

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-WM-01 | Sistem harus menampilkan data cuaca real-time area perairan STS Bunati melalui integrasi Weather API | System |
| FR-WM-02 | Sistem harus memberikan warning alert jika tinggi gelombang melebihi 2.5 meter | System |
| FR-WM-03 | Sistem harus memberikan critical alert dan rekomendasi penghentian operasi jika tinggi gelombang melebihi 3 meter | System |
| FR-WM-04 | Sistem harus menampilkan weather forecast H+24 dan H+48 untuk perencanaan operasi | LPS Operator |
| FR-WM-05 | Sistem harus mencatat log cuaca otomatis setiap interval waktu yang dikonfigurasi | System |
| FR-WM-06 | LPS Operator harus dapat mengirimkan notifikasi cuaca ke seluruh kapal di area perairan secara massal | LPS Operator |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | Full |
| VHF Operator | View |
| Security Operator | View |
| Admin Kepelabuhan | Full |
| KSOP | View |
| Bea Cukai | — |
| Customer | View |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M5](../BRD-LPS-V3-Analysis.md) & [Section 3.4.5](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M5)*
- Modul terkait: [M4 — Incident & Emergency](m4-incident-emergency.md), [M10 — Customer Dashboard](m10-customer-dashboard.md)
