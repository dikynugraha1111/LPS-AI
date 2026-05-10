# BRD Modul M11 — Monitoring & Visibility Dashboard

**Kategori:** Monitoring  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Catatan:** Modul ini menggantikan Reporting & Analytics (dipindahkan ke STS Platform).  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.11](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang menyediakan dashboard monitoring operasional real-time untuk seluruh stakeholder LPS. Modul ini menggantikan Reporting & Analytics yang telah dipindahkan ke STS Platform, dengan fokus khusus pada visualisasi data operasional secara real-time.

## Tujuan

- Menyediakan visibilitas operasional real-time kepada seluruh stakeholder.
- Menampilkan ringkasan status kapal, cuaca, insiden, dan komunikasi dalam satu dashboard terpadu.
- Mendukung pengambilan keputusan operasional yang cepat oleh LPS Operator.

## Cakupan (In Scope)

- Dashboard utama: jumlah kapal aktif, status cuaca, insiden aktif, status anchor point.
- Peta STS area real-time dengan overlay posisi kapal, zona, dan anchor point.
- Panel recent activity: kapal masuk/keluar, alert terbaru, komunikasi terakhir.
- Status integrasi API pemerintah (Inaportnet, SIMoPEL, Bea Cukai) — last sync status.
- Weather widget real-time dengan indikator level bahaya.
- Akses dashboard sesuai role (KSOP, Bea Cukai, Pemilik Kapal mendapat view terbatas sesuai hak akses masing-masing).

> **Catatan:** Laporan periodik (harian/bulanan/tahunan), laporan billing, dan export PDF/Excel dikelola oleh STS Platform melalui modul Operational & Financial Reporting.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-MD-01 | Dashboard harus menampilkan jumlah dan posisi kapal aktif real-time di area STS beserta status (Verified/Unknown) | Admin |
| FR-MD-02 | Dashboard harus menampilkan status cuaca real-time dengan indikator level bahaya (Normal/Warning/Critical) | LPS Operator |
| FR-MD-03 | Dashboard harus menampilkan status insiden aktif dan progres penanganan | LPS Operator |
| FR-MD-04 | Dashboard harus menampilkan status anchor point (Available/Occupied) secara real-time | Admin |
| FR-MD-05 | Dashboard harus menampilkan recent activity: kapal masuk/keluar area, alert terbaru, komunikasi VHF terakhir | LPS Operator |
| FR-MD-06 | Dashboard harus menampilkan status koneksi API eksternal (AIS, Weather, Inaportnet, SIMoPEL, Bea Cukai, STS Platform) | Admin |
| FR-MD-07 | KSOP, Bea Cukai, dan Customer harus dapat mengakses dashboard sesuai hak akses masing-masing (view-only) | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | Full |
| VHF Operator | View |
| Security Operator | View |
| Admin Kepelabuhan | Full |
| KSOP | View |
| Bea Cukai | View |
| Customer | View (terbatas) |
| SuperAdmin | Full |

---

## Referensi

- BRD Utama: [Section 2.1 M11](../BRD-LPS-V3-Analysis.md) & [Section 3.4.11](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M11)*
- Modul terkait: [M1 — Vessel Monitoring](m1-vessel-monitoring.md), [M4 — Incident](m4-incident-emergency.md), [M5 — Weather](m5-weather-monitoring.md)
