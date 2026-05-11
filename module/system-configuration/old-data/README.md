# M9 — Master Data & Configuration

## Overview

Modul fondasi yang menyimpan seluruh data referensi dan konfigurasi sistem LPS yang digunakan oleh modul-modul lainnya. Semua data master harus diisi sebelum modul operasional dapat digunakan. M9 adalah prasyarat bagi seluruh modul operasional.

**PIC:** BA Eksternal 2
**Kategori:** Foundation

**Primary Actors:** Admin Kepelabuhan, Finance, SuperAdmin

## Assignment

| Role | Tanggung Jawab |
|---|---|
| Admin Kepelabuhan | Vessel Master, Stakeholder Master, Zone & Anchor Point, Equipment Master |
| Finance | Rate Card Management |
| SuperAdmin | User & Role Management, System Parameter Configuration, API Configuration |

## Scope Focus

### Data Master

- Vessel Master: MV, Tug Boat, Barge, data identitas kapal (IMO, Call Sign, Flag)
- Stakeholder Master: KSOP, Bea Cukai, BUP, Pemilik Kapal, Badan Usaha Pemanduan
- Zona Perairan: batas area konsesi, zona larangan, titik berlabuh
- Rate Card layanan kepelabuhan
- Equipment Master: AIS Station, VHF Radio, RADAR, peralatan Oil Spill Response

### Configuration

- User & Role Management
- Weather threshold configuration (gelombang, angin, visibilitas)
- Alert & Notifikasi configuration
- Integrasi API configuration (Inaportnet, SIMoPEL, Weather, AIS Provider)

## Dependencies

| Modul | Dependency |
|---|---|
| None | M9 adalah modul fondasi. Semua modul lain bergantung pada M9. |

M9 menyediakan data referensi untuk:

- M1 (Vessel Monitoring): Vessel Master, Zone & Anchor Point, API Configuration
- M2 (Vessel Communication): Vessel Master, Equipment Master (VHF)
- M3 (Government Integration): Vessel Master, Stakeholder Master, API Configuration
- M4 (Incident & Emergency): Vessel Master, Equipment Master
- M5 (Weather Monitoring): System Config (threshold), API Configuration
- M6 (Radar & Navigation): Vessel Master, Zone & Anchor Point, Equipment Master (RADAR)
- M7 (Billing & Service Charge): Vessel Master, Stakeholder Master, Rate Card
- M8 (Reporting & Analytics): Semua data master sebagai referensi laporan

## Sub-modules

| Sub-module | FSD Section | Deskripsi |
|---|---|---|
| Vessel Master | 4.1.1 | CRUD data kapal dengan validasi IMO/MMSI, import CSV, auto-flag AIS |
| Stakeholder Master | 4.1.2 | Data perusahaan dan pihak terlibat, relasi 1-to-many ke Vessel |
| Zone & Anchor Point Configuration | 4.1.3 | Konfigurasi zona perairan (polygon) dan titik jangkar (koordinat) |
| Rate Card Management | 4.1.4 | Tarif layanan dengan effective date, approval workflow, never-delete policy |
| User & Role Management | 4.1.5 | RBAC dengan 9 role, permission matrix, audit trail |
| Equipment Master & API Configuration | 4.1.6 | Data perangkat keras LPS, konfigurasi koneksi API eksternal |

## Deliverables

- Data model dan migration untuk semua entitas master
- REST API endpoints untuk CRUD semua entitas
- Frontend settings pages untuk manajemen data
- Audit trail pada semua perubahan data
- Seed data untuk roles, permissions, dan system config defaults
- CSV import untuk Vessel Master

## Timeline

M9 harus selesai sebelum modul operasional manapun dapat dikembangkan. Merupakan bagian dari milestone "Core Operasi & Master Data Completion".

## Related Documents

| Dokumen | Deskripsi |
|---|---|
| docs/designs/m9-master-data.md | Design document lengkap: data model, API design, frontend, business rules |
| docs/plans/m9-master-data-implementation.md | Implementation plan dengan step sequence |
| docs/handoffs/m9-step01-models-and-migration.md | Handoff step 1: Models & Migration |
| docs/handoffs/m9-step02-audit-and-user-role.md | Handoff step 2: Audit & User Role |
| docs/handoffs/m9-step03-vessel-and-stakeholder.md | Handoff step 3: Vessel & Stakeholder |
| docs/handoffs/m9-step04-zone-anchor-equipment.md | Handoff step 4: Zone, Anchor, Equipment |
| docs/handoffs/m9-step05-ratecard-apiconfig-sysconfig.md | Handoff step 5: Rate Card, API Config, System Config |
| docs/handoffs/m9-step06-frontend-settings.md | Handoff step 6: Frontend Settings |

## Module Files

| File | Deskripsi |
|---|---|
| [Requirements](requirements.md) | Functional requirements FR-MD-01 s/d FR-MD-07 dari BRD 3.3.8 |
| [Specifications](specifications.md) | Spesifikasi fungsional detail dari FSD 4.1 |
| [User Stories](user-stories.md) | 8 user stories (US-MD-001 s/d US-MD-008) dengan acceptance criteria |
