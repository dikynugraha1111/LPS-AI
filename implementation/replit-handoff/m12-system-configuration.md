# Replit Handoff — M12: System Configuration

## Context

Modul ini mengelola konfigurasi sistem LPS: user & role management, system parameters, API integration config, equipment management, audit log, dan STS data sync. Modul ini adalah **fondasi admin** — sebagian besar sub-modul ini (user, role, equipment) kemungkinan sudah sebagian diimplementasikan di admin web yang berjalan saat ini. Baca seluruh handoff ini dan **sesuaikan dengan kondisi aktual Replit** — jangan overwrite yang sudah ada.

## Prerequisites

- Admin auth (login admin, JWT/session admin) sudah berjalan.
- Tabel `admin_users` atau setaranya sudah ada (dari implementasi admin sebelumnya).
- Identifikasi tabel dan struktur yang sudah ada sebelum menjalankan migration.

## Tech Stack

- **Frontend:** React 19, Vite, TailwindCSS v4, shadcn/ui, wouter, TanStack React Query
- **Backend:** Go 1.25, jasoet/pkg/v2 (Echo, GORM, zerolog)
- **Database:** PostgreSQL, golang-migrate

---

## Step 1: Database Migration — Core Tables

File: `migrations/XXXXXX_create_m12_system_configuration.up.sql`

> **Penting:** Cek tabel yang sudah ada. Jalankan hanya bagian yang belum ada.

```sql
-- Roles (skip jika sudah ada)
CREATE TABLE IF NOT EXISTS roles (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name         VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100) NOT NULL,
    description  TEXT,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Role permissions per module
CREATE TABLE IF NOT EXISTS role_permissions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id    UUID NOT NULL REFERENCES roles(id),
    module     VARCHAR(50) NOT NULL,
    can_view   BOOLEAN NOT NULL DEFAULT FALSE,
    can_create BOOLEAN NOT NULL DEFAULT FALSE,
    can_edit   BOOLEAN NOT NULL DEFAULT FALSE,
    can_delete BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (role_id, module)
);

-- System config key-value store
CREATE TABLE IF NOT EXISTS system_configs (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_key   VARCHAR(100) NOT NULL UNIQUE,
    config_value TEXT NOT NULL,
    value_type   VARCHAR(20) NOT NULL DEFAULT 'string',
    description  TEXT,
    updated_by   UUID,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- External API integration configs
CREATE TABLE IF NOT EXISTS api_configs (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_name     VARCHAR(50) NOT NULL UNIQUE,
    display_name         VARCHAR(100) NOT NULL,
    base_url             TEXT,
    api_key_encrypted    TEXT,
    api_secret_encrypted TEXT,
    extra_params         JSONB,
    is_active            BOOLEAN NOT NULL DEFAULT TRUE,
    last_test_at         TIMESTAMPTZ,
    last_test_status     VARCHAR(20) DEFAULT 'UNTESTED',
    updated_by           UUID,
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert notification configs
CREATE TABLE IF NOT EXISTS alert_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(100) NOT NULL UNIQUE,
    is_enabled      BOOLEAN NOT NULL DEFAULT TRUE,
    recipient_roles TEXT[] NOT NULL DEFAULT '{}',
    updated_by      UUID,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Equipment master
CREATE TABLE IF NOT EXISTS equipment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    equipment_type      VARCHAR(50) NOT NULL,
    serial_number       VARCHAR(100),
    purchase_date       DATE,
    warranty_expiry     DATE,
    location            VARCHAR(255),
    status              VARCHAR(30) NOT NULL DEFAULT 'OPERATIONAL',
    next_maintenance_at DATE,
    notes               TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Equipment status history (immutable)
CREATE TABLE IF NOT EXISTS equipment_status_history (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id UUID NOT NULL REFERENCES equipment(id),
    old_status   VARCHAR(30),
    new_status   VARCHAR(30) NOT NULL,
    changed_by   UUID,
    notes        TEXT,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log (immutable, insert-only)
CREATE TABLE IF NOT EXISTS audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id    UUID,
    actor_name  VARCHAR(255),
    action      VARCHAR(50) NOT NULL,
    entity_type VARCHAR(100) NOT NULL,
    entity_id   VARCHAR(255),
    details     JSONB,
    ip_address  VARCHAR(45),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_audit_logs_actor_id    ON audit_logs(actor_id);
CREATE INDEX IF NOT EXISTS idx_audit_logs_entity_type ON audit_logs(entity_type);
CREATE INDEX IF NOT EXISTS idx_audit_logs_created_at  ON audit_logs(created_at DESC);

-- STS local cache — vessel
CREATE TABLE IF NOT EXISTS sts_vessels_cache (
    id           UUID PRIMARY KEY,
    name         VARCHAR(255) NOT NULL,
    imo_number   VARCHAR(20),
    mmsi         VARCHAR(20),
    vessel_type  VARCHAR(50),
    flag         VARCHAR(100),
    grt          DECIMAL(10,2),
    loa          DECIMAL(10,2),
    owner_name   VARCHAR(255),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    synced_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- STS sync log
CREATE TABLE IF NOT EXISTS sts_sync_log (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type    VARCHAR(50) NOT NULL,
    status         VARCHAR(20) NOT NULL,
    records_synced INTEGER DEFAULT 0,
    error_message  TEXT,
    synced_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Step 2: Seed Data

File: `migrations/XXXXXX_seed_m12_defaults.up.sql`

```sql
-- Seed roles (skip existing)
INSERT INTO roles (name, display_name, description) VALUES
    ('LPS_OPERATOR',      'LPS Operator',       'Monitoring peta AIS, komunikasi kapal, laporan insiden, weather alert'),
    ('VHF_OPERATOR',      'VHF Operator',       'Channel management, log komunikasi VHF, broadcast notifikasi'),
    ('SECURITY_OPERATOR', 'Security Operator',  'Deteksi kapal asing, checklist darurat, oil spill management'),
    ('ADMIN_KEPELABUHAN', 'Admin Kepelabuhan',  'Semua modul operasional, approval, monitoring integrasi pemerintah'),
    ('FINANCE',           'Finance',            'Kalkulasi biaya, approval invoice, payment reconciliation, rate card'),
    ('KSOP',              'KSOP',               'View-only: monitoring kapal, laporan, insiden, status PKK'),
    ('BEA_CUKAI',         'Bea Cukai',          'View-only: manifes muatan, dokumen kepabeanan'),
    ('CUSTOMER',          'Customer',           'Customer Portal: nominasi kapal, status, EPB & invoice'),
    ('SUPER_ADMIN',       'SuperAdmin',         'CRUD user, role, konfigurasi sistem, API config, audit trail penuh')
ON CONFLICT (name) DO NOTHING;

-- Seed system_configs defaults
INSERT INTO system_configs (config_key, config_value, value_type, description) VALUES
    ('weather.wave_warning_threshold',       '1.5',   'number',  'Gelombang warning level (meter)'),
    ('weather.wave_critical_threshold',      '2.5',   'number',  'Gelombang critical level (meter)'),
    ('weather.wind_warning_threshold',       '15',    'number',  'Angin warning level (knot)'),
    ('weather.wind_critical_threshold',      '25',    'number',  'Angin critical level (knot)'),
    ('weather.visibility_warning_threshold', '2',     'number',  'Visibilitas warning level (km)'),
    ('weather.visibility_critical_threshold','1',     'number',  'Visibilitas critical level (km)'),
    ('weather.update_interval_minutes',      '15',    'number',  'Interval update data cuaca (menit)'),
    ('data.retention_years',                 '5',     'number',  'Retensi data operasional (tahun)'),
    ('ais.message_template_default',         'Perhatian semua kapal di area STS Bunati:', 'string', 'Template default pesan broadcast AIS')
ON CONFLICT (config_key) DO NOTHING;

-- Seed api_configs
INSERT INTO api_configs (integration_name, display_name, is_active) VALUES
    ('INAPORTNET',   'Inaportnet (Kemenhub)',    TRUE),
    ('SIMODEL',      'SIMoPEL (Kemenhub)',        TRUE),
    ('WEATHER_API',  'Weather API',               TRUE),
    ('AIS_PROVIDER', 'AIS Data Provider',         TRUE),
    ('STS_PLATFORM', 'STS Platform API',          TRUE),
    ('BEA_CUKAI',    'Bea Cukai API',             TRUE)
ON CONFLICT (integration_name) DO NOTHING;

-- Seed alert_configs
INSERT INTO alert_configs (event_type, is_enabled, recipient_roles) VALUES
    ('WEATHER_WARNING',  TRUE, ARRAY['LPS_OPERATOR', 'ADMIN_KEPELABUHAN']),
    ('WEATHER_CRITICAL', TRUE, ARRAY['LPS_OPERATOR', 'ADMIN_KEPELABUHAN', 'SUPER_ADMIN']),
    ('VESSEL_ARRIVAL',   TRUE, ARRAY['LPS_OPERATOR', 'ADMIN_KEPELABUHAN']),
    ('INCIDENT',         TRUE, ARRAY['LPS_OPERATOR', 'SECURITY_OPERATOR', 'ADMIN_KEPELABUHAN', 'SUPER_ADMIN']),
    ('PAYMENT_REMINDER', TRUE, ARRAY['FINANCE', 'ADMIN_KEPELABUHAN']),
    ('SYSTEM_ALERT',     TRUE, ARRAY['SUPER_ADMIN'])
ON CONFLICT (event_type) DO NOTHING;
```

---

## Step 3: Backend — Models

File: `internal/sysconfig/model.go`

```go
type Role struct {
    ID          uuid.UUID `gorm:"type:uuid;primaryKey"`
    Name        string    `gorm:"uniqueIndex;not null"`
    DisplayName string    `gorm:"not null"`
    Description *string
    IsActive    bool      `gorm:"not null;default:true"`
    CreatedAt   time.Time
    UpdatedAt   time.Time
    Permissions []RolePermission `gorm:"foreignKey:RoleID"`
}

type RolePermission struct {
    ID        uuid.UUID `gorm:"type:uuid;primaryKey"`
    RoleID    uuid.UUID `gorm:"not null"`
    Module    string    `gorm:"not null"`
    CanView   bool      `gorm:"not null;default:false"`
    CanCreate bool      `gorm:"not null;default:false"`
    CanEdit   bool      `gorm:"not null;default:false"`
    CanDelete bool      `gorm:"not null;default:false"`
}

type SystemConfig struct {
    ID          uuid.UUID  `gorm:"type:uuid;primaryKey"`
    ConfigKey   string     `gorm:"uniqueIndex;not null"`
    ConfigValue string     `gorm:"not null"`
    ValueType   string     `gorm:"not null;default:'string'"`
    Description *string
    UpdatedBy   *uuid.UUID
    UpdatedAt   time.Time
}

type ApiConfig struct {
    ID                  uuid.UUID  `gorm:"type:uuid;primaryKey"`
    IntegrationName     string     `gorm:"uniqueIndex;not null"`
    DisplayName         string     `gorm:"not null"`
    BaseURL             *string
    ApiKeyEncrypted     *string
    ApiSecretEncrypted  *string
    ExtraParams         datatypes.JSON
    IsActive            bool       `gorm:"not null;default:true"`
    LastTestAt          *time.Time
    LastTestStatus      string     `gorm:"default:'UNTESTED'"`
    UpdatedBy           *uuid.UUID
    UpdatedAt           time.Time
}

type AlertConfig struct {
    ID             uuid.UUID      `gorm:"type:uuid;primaryKey"`
    EventType      string         `gorm:"uniqueIndex;not null"`
    IsEnabled      bool           `gorm:"not null;default:true"`
    RecipientRoles pq.StringArray `gorm:"type:text[]"`
    UpdatedBy      *uuid.UUID
    UpdatedAt      time.Time
}

type Equipment struct {
    ID                uuid.UUID  `gorm:"type:uuid;primaryKey"`
    Name              string     `gorm:"not null"`
    EquipmentType     string     `gorm:"not null"`
    SerialNumber      *string
    PurchaseDate      *time.Time
    WarrantyExpiry    *time.Time
    Location          *string
    Status            string     `gorm:"not null;default:'OPERATIONAL'"`
    NextMaintenanceAt *time.Time
    Notes             *string
    IsActive          bool       `gorm:"not null;default:true"`
    CreatedAt         time.Time
    UpdatedAt         time.Time
    StatusHistory     []EquipmentStatusHistory `gorm:"foreignKey:EquipmentID"`
}

type EquipmentStatusHistory struct {
    ID          uuid.UUID  `gorm:"type:uuid;primaryKey"`
    EquipmentID uuid.UUID  `gorm:"not null"`
    OldStatus   *string
    NewStatus   string     `gorm:"not null"`
    ChangedBy   *uuid.UUID
    Notes       *string
    ChangedAt   time.Time
}

type AuditLog struct {
    ID         uuid.UUID      `gorm:"type:uuid;primaryKey"`
    ActorID    *uuid.UUID
    ActorName  *string
    Action     string         `gorm:"not null"`
    EntityType string         `gorm:"not null"`
    EntityID   *string
    Details    datatypes.JSON
    IPAddress  *string
    CreatedAt  time.Time
}
```

---

## Step 4: Backend — Audit Log Helper

File: `internal/sysconfig/audit.go`

Buat helper yang dipanggil dari semua handler yang butuh logging:

```go
func WriteAuditLog(db *gorm.DB, actorID *uuid.UUID, actorName, action, entityType, entityID string, details interface{}, ip string) {
    detailsJSON, _ := json.Marshal(details)
    log := AuditLog{
        ID:         uuid.New(),
        ActorID:    actorID,
        ActorName:  &actorName,
        Action:     action,
        EntityType: entityType,
        EntityID:   &entityID,
        Details:    datatypes.JSON(detailsJSON),
        IPAddress:  &ip,
        CreatedAt:  time.Now(),
    }
    db.Create(&log)
}
```

---

## Step 5: Backend — API Config Encryption

File: `internal/sysconfig/crypto.go`

API key dan secret dienkripsi sebelum disimpan ke DB. Gunakan AES-256-GCM dengan key dari env var `ENCRYPTION_KEY` (32 byte hex).

```go
func Encrypt(plaintext, keyHex string) (string, error) { ... }
func Decrypt(ciphertext, keyHex string) (string, error) { ... }
func MaskValue(s string) string {
    if len(s) <= 4 { return "****" }
    return s[:2] + strings.Repeat("*", len(s)-4) + s[len(s)-2:]
}
```

Saat response API config ke frontend: **jangan kembalikan** `api_key_encrypted` atau `api_secret_encrypted`. Kembalikan field `api_key_masked` (hasil `MaskValue`) dan boolean `has_api_key`.

---

## Step 6: Backend — Handlers

File: `internal/sysconfig/handler.go`

Semua route di bawah prefix `/api/admin/settings/` dengan middleware admin JWT.

### User Management

```
GET  /api/admin/settings/users          — list users (query: role, status, search)
POST /api/admin/settings/users          — create user (hash password, audit log)
GET  /api/admin/settings/users/:id      — get user detail
PUT  /api/admin/settings/users/:id      — update user (audit log sebelum/sesudah)
PUT  /api/admin/settings/users/:id/deactivate — nonaktifkan (cek tidak deactivate diri sendiri)
PUT  /api/admin/settings/users/:id/activate   — aktifkan
```

### Role & Permission

```
GET  /api/admin/settings/roles                    — list roles
GET  /api/admin/settings/roles/:id/permissions    — get permission matrix
PUT  /api/admin/settings/roles/:id/permissions    — update permission matrix (upsert per module)
```

### System Config

```
GET  /api/admin/settings/system-configs           — list all key-value pairs
PUT  /api/admin/settings/system-configs           — bulk update (array of {key, value})
```

### API Integrations

```
GET  /api/admin/settings/api-configs              — list all (mask api_key fields)
PUT  /api/admin/settings/api-configs/:name        — update config (encrypt key/secret before save)
POST /api/admin/settings/api-configs/:name/test   — test connection, update last_test_at + last_test_status
```

Test connection logic:
- `INAPORTNET`, `SIMODEL`, `BEA_CUKAI`: HTTP GET ke `base_url/health` atau `base_url/ping` dengan Authorization header, timeout 5 detik.
- `WEATHER_API`, `AIS_PROVIDER`: sama.
- `STS_PLATFORM`: HTTP GET ke `base_url/api/health`.
- Update `last_test_at` dan `last_test_status` (OK/FAILED) terlepas dari hasil test.

### Alert Config

```
GET  /api/admin/settings/alert-configs            — list all alert configs
PUT  /api/admin/settings/alert-configs            — bulk update
```

### Equipment

```
GET  /api/admin/settings/equipment                — list (query: type, status)
POST /api/admin/settings/equipment                — create
GET  /api/admin/settings/equipment/:id            — detail + status history
PUT  /api/admin/settings/equipment/:id            — update data
PUT  /api/admin/settings/equipment/:id/status     — update status (insert equipment_status_history, cek critical alert)
```

Pada status update: jika `new_status != OPERATIONAL` dan `equipment_type` adalah `AIS_STATION`, `RADAR`, atau `VHF_RADIO` → log alert ke audit_logs dengan `action = SYSTEM_ALERT`, `entity_type = EQUIPMENT`.

### Audit Log

```
GET  /api/admin/settings/audit-logs               — list (query: actor_id, entity_type, action, from, to, page, limit)
```

Tidak ada endpoint POST/PUT/DELETE untuk audit_logs.

### STS Sync

```
GET  /api/admin/settings/sts-sync/status          — status terakhir per entity dari sts_sync_log
POST /api/admin/settings/sts-sync/trigger         — trigger manual sync (panggil STS Platform API, upsert sts_vessels_cache, insert sts_sync_log)
```

---

## Step 7: Frontend — Settings Layout

File: `src/pages/admin/settings/SettingsLayout.tsx`

Buat layout dengan sidebar navigasi settings:

```
Settings
├── User Management       → /admin/settings/users
├── Role & Permissions    → /admin/settings/roles
├── System Parameters     → /admin/settings/system
├── API Integrations      → /admin/settings/api-integrations
├── Equipment             → /admin/settings/equipment
├── Audit Log             → /admin/settings/audit-logs
└── STS Data Sync         → /admin/settings/sts-sync
```

---

## Step 8: Frontend — User Management Page

File: `src/pages/admin/settings/UserManagementPage.tsx`

- Table: nama, email, role (badge), status (badge Active/Inactive), last login, actions
- Search bar + filter by role dan status
- "Tambah User" button → modal form atau navigate ke `/admin/settings/users/new`
- Row actions: Edit, Activate/Deactivate (tergantung status saat ini)
- Konfirmasi dialog sebelum deactivate

File: `src/pages/admin/settings/UserFormPage.tsx`

Fields: Name, Email, Username, Password (hanya saat create), Role (dropdown dari `/api/admin/settings/roles`), Active status

---

## Step 9: Frontend — Role & Permission Page

File: `src/pages/admin/settings/RolePermissionsPage.tsx`

- Dropdown atau tab untuk memilih role
- Tabel permission matrix: baris = modul (M1–M12), kolom = View / Create / Edit / Delete (checkbox)
- "Simpan" button → `PUT /api/admin/settings/roles/:id/permissions`
- Show info: "Perubahan permission berlaku pada login berikutnya pengguna."

---

## Step 10: Frontend — System Config Page

File: `src/pages/admin/settings/SystemConfigPage.tsx`

Kelompokkan config ke dalam section:

**Weather Threshold:**
- Gelombang Warning/Critical (number input, satuan: meter)
- Angin Warning/Critical (number input, satuan: knot)
- Visibilitas Warning/Critical (number input, satuan: km)
- Interval update cuaca (number input, satuan: menit)

**Data Retention:**
- Retensi data operasional (number input, satuan: tahun)

**AIS:**
- Template pesan default (textarea)

"Simpan Semua" button → konfirmasi dialog → `PUT /api/admin/settings/system-configs`

---

## Step 11: Frontend — API Integrations Page

File: `src/pages/admin/settings/ApiIntegrationsPage.tsx`

- Card per integrasi: nama, status badge (Active/Inactive), status koneksi terakhir (OK/FAILED/UNTESTED), waktu test terakhir
- "Edit" button → expand inline form atau modal:
  - Base URL (text input)
  - API Key (text input, tampilkan mask `****` jika sudah ada, placeholder "Kosongkan untuk tidak mengubah")
  - API Secret (sama)
  - Extra Params (textarea JSON, opsional)
  - Toggle Active/Inactive
- "Test Connection" button → loading spinner → tampilkan hasil (OK/FAILED) dengan waktu response
- Jangan tampilkan nilai API key; submit form: jika field API key kosong, server tidak overwrite yang sudah ada

---

## Step 12: Frontend — Equipment Page

File: `src/pages/admin/settings/EquipmentPage.tsx`

- Table: nama, tipe (badge), serial number, status (badge dengan warna), lokasi, next maintenance, actions
- Filter by tipe dan status
- "Tambah Equipment" button

File: `src/pages/admin/settings/EquipmentDetailPage.tsx`

- Detail equipment + timeline status history
- "Ubah Status" button → dropdown status + notes input + konfirmasi

Status badge colors:
- `OPERATIONAL` → green
- `MAINTENANCE` → yellow
- `STANDBY` → blue
- `OUT_OF_SERVICE` → red

---

## Step 13: Frontend — Audit Log Page

File: `src/pages/admin/settings/AuditLogPage.tsx`

- Table: timestamp, actor name, action (badge), entity type, entity ID, IP address
- Filter: entity type (dropdown), action (dropdown), actor (text search), dari tanggal, sampai tanggal
- Klik baris → expand detail dengan JSON diff sebelum/sesudah (rendered sebagai formatted JSON)
- Pagination

---

## Step 14: Frontend — STS Sync Page

File: `src/pages/admin/settings/StsSyncPage.tsx`

- Status card per entity type (Vessel, Stakeholder, Zone, Anchor Point):
  - Status badge (SUCCESS/FAILED/PARTIAL)
  - Records synced
  - Timestamp sync terakhir
- "Trigger Manual Sync" button → loading state → refresh status cards
- Jika koneksi STS tidak aktif (`api_configs.STS_PLATFORM.is_active = false`): tampilkan warning "STS Platform API tidak aktif. Aktifkan di halaman API Integrations."

---

## Acceptance Checklist

**User & Role Management**
- [ ] SuperAdmin dapat membuat, mengedit, mengaktifkan, dan menonaktifkan user
- [ ] SuperAdmin tidak dapat menonaktifkan akun sendiri (tampilkan error informatif)
- [ ] Username dan email yang duplikat ditolak dengan pesan error
- [ ] Semua perubahan user tercatat di audit log
- [ ] SuperAdmin dapat melihat dan mengubah permission matrix per role per modul

**System Configuration**
- [ ] Weather threshold, alert config, data retention, AIS template dapat diubah dan berlaku langsung
- [ ] Semua perubahan konfigurasi memerlukan konfirmasi sebelum disimpan
- [ ] Semua perubahan konfigurasi tercatat di audit log

**API Integrations**
- [ ] Semua integrasi API ditampilkan dengan status koneksi terakhir
- [ ] API key dan secret ditampilkan sebagai masked (`****`), tidak pernah dikembalikan plaintext
- [ ] "Test Connection" bekerja dan memperbarui status koneksi
- [ ] Mengosongkan field API key saat update tidak menghapus nilai yang sudah tersimpan

**Equipment Management**
- [ ] Admin dapat membuat, memperbarui data, dan mengubah status equipment
- [ ] History perubahan status tersimpan dan tidak dapat dihapus
- [ ] Alert tercatat di audit log ketika peralatan kritis (AIS, RADAR, VHF) menjadi non-Operational
- [ ] Equipment list dapat difilter by tipe dan status

**Audit Log**
- [ ] Audit log dapat difilter by entity type, actor, action, dan rentang waktu
- [ ] Detail perubahan (sebelum/sesudah) ditampilkan per entry
- [ ] Tidak ada tombol atau endpoint untuk mengedit/menghapus audit log

**STS Data Sync**
- [ ] Status dan timestamp sync terakhir ditampilkan per entity type
- [ ] Manual trigger sync berjalan dan memperbarui sts_sync_log
- [ ] Warning ditampilkan jika STS Platform API tidak aktif
- [ ] LPS tetap menampilkan data dari local cache saat STS tidak tersedia
