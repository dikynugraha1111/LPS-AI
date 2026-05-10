# BRD Modul M12 — System Configuration

**Kategori:** Konfigurasi  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Catatan:** Sub-modul yang tersisa dari Master Data & Configuration lama setelah data master dipindahkan ke STS Platform.  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.12](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Sub-modul yang mengelola konfigurasi sistem LPS, pengelolaan user dan role, serta parameter operasional. Modul ini merupakan sisa dari Master Data & Configuration sebelumnya, setelah data master entity (vessel, stakeholder, rate card) dipindahkan ke STS Platform.

## Tujuan

- Menyediakan pengelolaan user, role, dan permission untuk akses LPS.
- Menyediakan konfigurasi parameter sistem (weather threshold, alert config, API keys).
- Menyediakan audit trail untuk seluruh aktivitas sistem.

## Cakupan (In Scope)

### User & Role Management
- CRUD user dengan role assignment.
- Permission matrix per role dan per modul.
- Session management dan security policy.

### System Configuration
- Weather threshold configuration (gelombang, angin, visibilitas).
- Alert & notifikasi configuration.
- Integrasi API configuration (Inaportnet, SIMoPEL, Weather API, AIS Provider, **STS Platform API**).
- AIS message template configuration.
- Data retention parameter configuration.

### Equipment Management
- CRUD data equipment LPS (AIS Station, VHF Radio, RADAR, CCTV, Weather Station).
- Status operasional equipment: Operational, Maintenance, Out of Service, Standby.
- Jadwal next maintenance.

### Audit Log
- Immutable log seluruh aktivitas sistem.
- Filter berdasarkan entity type, actor, dan rentang waktu.

> **Catatan:** Data master vessel, stakeholder, zona perairan, anchor point, dan rate card dikelola oleh STS Platform dan dikonsumsi oleh LPS melalui API sinkronisasi. LPS menyimpan local cache data master untuk operasional, dengan mekanisme sync berkala. Registrasi customer baru di LPS Portal akan disinkronisasi ke STS Platform secara otomatis.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-SC-01 | SuperAdmin harus dapat mengelola user, role, dan permission | SuperAdmin |
| FR-SC-02 | SuperAdmin harus dapat mengkonfigurasi parameter sistem (weather threshold, alert config, API keys, AIS message templates, data retention) | SuperAdmin |
| FR-SC-03 | Admin harus dapat mengelola data equipment LPS (AIS Station, VHF Radio, RADAR) beserta status operasionalnya | Admin |
| FR-SC-04 | Sistem harus menyimpan seluruh aktivitas dalam immutable audit log yang dapat difilter berdasarkan entity type, actor, dan rentang waktu | System |
| FR-SC-05 | Sistem harus melakukan sinkronisasi data master (vessel, stakeholder, zona, anchor point) dari STS Platform secara berkala melalui API | System |
| FR-SC-06 | Sistem harus menyimpan local cache data master dari STS Platform untuk memastikan operasional berjalan meskipun koneksi ke STS terputus sementara | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | — |
| VHF Operator | — |
| Security Operator | — |
| Admin Kepelabuhan | Limited |
| KSOP | — |
| Bea Cukai | — |
| Customer | — |
| SuperAdmin | Full |

---

## Referensi

- BRD Utama: [Section 2.1 M12](../BRD-LPS-V3-Analysis.md) & [Section 3.4.12](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M12)*
- Appendix BRD: [A.1 Master Data (Dipindahkan)](../BRD-LPS-V3-Analysis.md)
