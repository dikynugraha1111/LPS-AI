# M9 — Requirements

Functional requirements untuk modul Master Data & Configuration, diambil dari BRD section 3.3.8.

## Functional Requirements

| FR ID | Requirement | Actor |
|---|---|---|
| FR-MD-01 | Admin harus dapat mengelola data vessel (CRUD): MV, Tug Boat, Barge beserta detail teknis | Admin |
| FR-MD-02 | Admin harus dapat mengelola data stakeholder (CRUD): KSOP, Bea Cukai, BUP, Pemilik Kapal, Badan Usaha Pemanduan | Admin |
| FR-MD-03 | Admin harus dapat mengelola zona perairan (CRUD): batas area konsesi, zona berlabuh, zona larangan | Admin |
| FR-MD-04 | Finance harus dapat mengelola rate card layanan kepelabuhan dengan effective date | Finance |
| FR-MD-05 | Admin harus dapat mengelola data equipment LPS (AIS Station, VHF Radio, RADAR) beserta status operasionalnya | Admin |
| FR-MD-06 | SuperAdmin harus dapat mengelola user, role, dan permission | SuperAdmin |
| FR-MD-07 | SuperAdmin harus dapat mengkonfigurasi parameter sistem (weather threshold, alert config, API keys) | SuperAdmin |

## Cakupan Data Master

| Entitas | Deskripsi | Pengelola |
|---|---|---|
| Vessel Master | MV, Tug Boat, Barge, data identitas kapal (IMO, Call Sign, Flag) | Admin Kepelabuhan |
| Stakeholder Master | KSOP, Bea Cukai, BUP, Pemilik Kapal, Badan Usaha Pemanduan | Admin Kepelabuhan |
| Zona Perairan | Batas area konsesi, zona larangan, titik berlabuh | Admin Kepelabuhan |
| Rate Card | Tarif layanan kepelabuhan | Finance |
| Equipment Master | AIS Station, VHF Radio, RADAR, peralatan Oil Spill Response | Admin Kepelabuhan |

## Cakupan Configuration

| Konfigurasi | Deskripsi | Pengelola |
|---|---|---|
| User & Role Management | RBAC dengan 9 role sistem | SuperAdmin |
| Weather Threshold | Konfigurasi batas gelombang, angin, visibilitas | SuperAdmin |
| Alert & Notifikasi | Konfigurasi event trigger dan penerima notifikasi | SuperAdmin |
| Integrasi API | Konfigurasi koneksi Inaportnet, SIMoPEL, Weather, AIS Provider | SuperAdmin |

## Traceability

| FR ID | User Story | FSD Section |
|---|---|---|
| FR-MD-01 | US-MD-001 | 4.1.1 Vessel Master |
| FR-MD-02 | US-MD-002 | 4.1.2 Stakeholder Master |
| FR-MD-03 | US-MD-003 | 4.1.3 Zone & Anchor Point Configuration |
| FR-MD-04 | US-MD-004 | 4.1.4 Rate Card Management |
| FR-MD-05 | US-MD-005, US-MD-007 | 4.1.5 User & Role Management, 4.1.6 Equipment Master |
| FR-MD-06 | US-MD-005, US-MD-006 | 4.1.5 User & Role Management |
| FR-MD-07 | US-MD-006, US-MD-008 | 4.1.6 Equipment Master & API Configuration |
