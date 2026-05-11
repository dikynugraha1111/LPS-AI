# M12 — System Configuration: Functional Requirements

Derived from BRD Section 3.4.12 and Section 2.1 Module 12.

## Functional Requirements

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-SC-01 | SuperAdmin harus dapat mengelola user, role, dan permission (CRUD user, role assignment, permission matrix per modul) | SuperAdmin | Must Have |
| FR-SC-02 | SuperAdmin harus dapat mengkonfigurasi parameter sistem: weather threshold (gelombang, angin, visibilitas), alert config, API keys (Inaportnet, SIMoPEL, Weather API, AIS Provider, STS Platform API), AIS message templates, dan data retention period | SuperAdmin | Must Have |
| FR-SC-03 | Admin Kepelabuhan harus dapat mengelola data equipment LPS (AIS Station, VHF Radio, RADAR, CCTV, Weather Station) beserta status operasionalnya (Operational, Maintenance, Out of Service, Standby) dan jadwal next maintenance | Admin | Must Have |
| FR-SC-04 | Sistem harus menyimpan seluruh aktivitas dalam immutable audit log yang dapat difilter berdasarkan entity type, actor, dan rentang waktu | System | Must Have |
| FR-SC-05 | Sistem harus melakukan sinkronisasi data master (vessel, stakeholder, zona, anchor point) dari STS Platform secara berkala melalui API | System | Must Have |
| FR-SC-06 | Sistem harus menyimpan local cache data master dari STS Platform untuk memastikan operasional berjalan meskipun koneksi ke STS terputus sementara | System | Must Have |

## Cakupan Sub-modul

### User & Role Management (FR-SC-01)

| Entitas | Aksi | Pengelola |
|---------|------|-----------|
| User account | CRUD, aktivasi/nonaktifkan | SuperAdmin |
| Role | Assignment ke user | SuperAdmin |
| Permission matrix | Konfigurasi per role per modul | SuperAdmin |

**System Roles (diwarisi dari implementasi admin yang sudah berjalan):**

| Role | Platform | Tipe |
|------|----------|------|
| LPS Operator | Web Admin | Internal TBK |
| VHF Operator | Web Admin | Internal TBK |
| Security Operator | Web Admin | Internal TBK |
| Admin Kepelabuhan | Web Admin | Internal TBK |
| Finance | Web Admin | Internal TBK |
| KSOP | Web Portal | Eksternal (Regulator) |
| Bea Cukai | Web Portal | Eksternal (Regulator) |
| Customer | Customer Portal | Eksternal (Customer) |
| SuperAdmin | Web Admin | Internal TBK |

### System Configuration (FR-SC-02)

| Parameter | Deskripsi |
|-----------|-----------|
| Weather threshold | Warning dan critical level untuk gelombang (m), angin (knot), visibilitas (km) |
| Alert config | Event trigger dan penerima notifikasi per event type |
| API keys | Endpoint URL, API key, dan parameter koneksi per integrasi eksternal |
| AIS message templates | Template pesan broadcast AIS |
| Data retention | Periode retensi per kategori data operasional |

### Equipment Management (FR-SC-03)

| Equipment Type | Keterangan |
|---------------|------------|
| AIS Station | AIS Base Station |
| VHF Radio | VHF Coastal Radio |
| RADAR System | Navigation RADAR |
| CCTV | Kamera pengawas area perairan |
| Weather Station | Sensor cuaca lokal |

### Audit Log (FR-SC-04)

- Insert-only, tidak dapat diedit atau dihapus
- Filter: entity type, actor (user), rentang waktu, action type

### STS Data Sync (FR-SC-05, FR-SC-06)

- Sinkronisasi berkala dari STS Platform API
- Entity yang di-cache: vessel, stakeholder, zona perairan, anchor point
- Fallback ke local cache saat koneksi STS terputus

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| Credential Security | BRD §3.5 | API key dan credential eksternal tersimpan terenkripsi, tidak pernah ditampilkan plaintext di UI |
| Audit Trail | BRD §3.5 | Semua perubahan data kritis masuk audit log dengan actor dan timestamp |
| No Hard Delete | BRD §3.5 | Tidak ada hard delete; data dinonaktifkan (soft delete) |
| Session Security | BRD §3.5 | Session timeout 30 menit tidak aktif; user nonaktif tidak bisa login |
| Data Retention | BRD §3.5 | Data operasional disimpan minimal 5 tahun |
