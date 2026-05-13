# M12 — System Configuration

## Overview

Modul yang mengelola konfigurasi sistem LPS, pengelolaan user dan role, serta parameter operasional. Merupakan sisa dari modul Master Data & Configuration lama setelah data master entity (vessel, stakeholder, rate card) dipindahkan ke STS Platform.

**Kategori:** Konfigurasi  
**BRD:** [Section 2.1 M12 & Section 3.4.12](../../document/BRD-LPS-V3-Analysis.md)  
**BRD per modul:** [document/brd/m12-system-configuration.md](../../document/brd/m12-system-configuration.md)

**Primary Actors:** SuperAdmin, Admin Kepelabuhan

---

## Boundaries

### In Scope

| Sub-modul | Deskripsi |
|-----------|-----------|
| User & Role Management | CRUD user, role assignment, permission matrix per modul, session policy |
| System Configuration | Weather threshold, alert config, API keys (Inaportnet, SIMoPEL, Weather, AIS, STS Platform), AIS message templates, data retention |
| Equipment Management | CRUD equipment LPS (AIS Station, VHF Radio, RADAR, CCTV, Weather Station), status operasional, jadwal maintenance |
| Audit Log | Immutable log seluruh aktivitas, filter by entity type / actor / rentang waktu |
| STS Data Sync | Trigger dan monitoring sinkronisasi local cache data master dari STS Platform |

### Out of Scope

- Data master vessel, stakeholder, zona perairan, anchor point, rate card → dikelola STS Platform
- Billing, invoice, PNBP → dikelola STS Platform
- Registrasi customer (Customer Portal) → dikelola M7

---

## Dependencies

| Modul | Jenis Dependency |
|-------|-----------------|
| **Tidak ada** | M12 adalah modul fondasi konfigurasi; tidak bergantung pada modul LPS lain |

M12 menyediakan konfigurasi untuk:

| Modul Konsumen | Yang Dikonsumsi dari M12 |
|----------------|--------------------------|
| M1 — Vessel Monitoring | API config (AIS Provider), local cache vessel & zone |
| M2 — Vessel Communication | Equipment config (VHF Radio), user roles |
| M3 — Government Integration | API config (Inaportnet, SIMoPEL, Bea Cukai) |
| M4 — Incident & Emergency | Equipment config (RADAR, Oil Spill), user roles |
| M5 — Weather Monitoring | Weather threshold config, API config (Weather API) |
| M6 — Radar & Navigation | Equipment config (RADAR), zone data (local cache) |
| M7–M10 — Customer Portal | Customer user roles, STS Platform API config |
| M11 — Monitoring Dashboard | User roles, audit log |

---

## Key Decisions

- **Data master tidak ada di LPS:** Vessel, stakeholder, zona, anchor point, rate card semuanya milik STS Platform. LPS hanya menyimpan **local cache** via sync berkala, agar tetap bisa beroperasi saat koneksi ke STS terputus sementara.
- **Warisan dari old-data (Master Data lama):** Sub-modul User & Role Management dan Equipment Master diwarisi langsung dari modul lama. Business rules utama (BR-MD-17 s/d BR-MD-25) tetap berlaku.
- **Audit log immutable:** Tidak ada soft/hard delete pada audit log. Dibuat insert-only.
- **API credential terenkripsi:** API key dan secret tidak pernah dikembalikan dalam bentuk plaintext dari API. Di-mask di UI setelah disimpan.

---

## Module Files

| File | Deskripsi |
|------|-----------|
| [README.md](README.md) | Boundaries, dependencies, key decisions |
| [requirements.md](requirements.md) | Functional requirements FR-SC-01 s/d FR-SC-06 |
| [specifications.md](specifications.md) | Spesifikasi teknis: DB schema, API endpoints, business rules |
| [user-stories.md](user-stories.md) | User stories US-SC-01 s/d US-SC-05 |

---

## UI/UX Design

**Surface B — Internal Operator** (English). Modul ini berada di area Settings dari sidebar Operator dengan sub-navigation nested.

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m12-system-configuration-ui.md`](../../implementation/design/m12-system-configuration-ui.md)

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
