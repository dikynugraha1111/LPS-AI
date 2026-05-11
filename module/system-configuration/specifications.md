# M12 — System Configuration: Technical Specifications

## 1. User & Role Management (FR-SC-01)

### 1.1 Business Rules

| Rule ID | Business Rule |
|---------|---------------|
| BR-SC-01 | Username dan email harus unik di seluruh sistem. |
| BR-SC-02 | Password: minimal 8 karakter, kombinasi huruf besar, huruf kecil, dan angka. |
| BR-SC-03 | User yang dinonaktifkan tidak dapat login; session aktif langsung diinvalidasi. |
| BR-SC-04 | Satu user hanya memiliki satu role aktif. Perubahan role efektif pada login berikutnya. |
| BR-SC-05 | SuperAdmin tidak dapat menonaktifkan akunnya sendiri. |
| BR-SC-06 | Semua perubahan user tersimpan di audit log beserta actor dan timestamp. |

### 1.2 Database Schema

```sql
CREATE TABLE roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(50) NOT NULL UNIQUE,
    -- LPS_OPERATOR | VHF_OPERATOR | SECURITY_OPERATOR | ADMIN_KEPELABUHAN
    -- FINANCE | KSOP | BEA_CUKAI | CUSTOMER | SUPER_ADMIN
    display_name VARCHAR(100) NOT NULL,
    description  TEXT,
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE admin_users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name         VARCHAR(255) NOT NULL,
    email        VARCHAR(255) NOT NULL UNIQUE,
    username     VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role_id      UUID NOT NULL REFERENCES roles(id),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role_permissions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id    UUID NOT NULL REFERENCES roles(id),
    module     VARCHAR(50) NOT NULL,
    -- M1 | M2 | M3 | M4 | M5 | M6 | M11 | M12
    can_view   BOOLEAN NOT NULL DEFAULT FALSE,
    can_create BOOLEAN NOT NULL DEFAULT FALSE,
    can_edit   BOOLEAN NOT NULL DEFAULT FALSE,
    can_delete BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (role_id, module)
);
```

### 1.3 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/admin/users | List all admin users | SuperAdmin |
| POST | /api/admin/users | Create new admin user | SuperAdmin |
| GET | /api/admin/users/:id | Get user detail | SuperAdmin |
| PUT | /api/admin/users/:id | Update user | SuperAdmin |
| PUT | /api/admin/users/:id/activate | Activate user | SuperAdmin |
| PUT | /api/admin/users/:id/deactivate | Deactivate user | SuperAdmin |
| GET | /api/admin/roles | List all roles | SuperAdmin |
| GET | /api/admin/roles/:id/permissions | Get permission matrix for role | SuperAdmin |
| PUT | /api/admin/roles/:id/permissions | Update permission matrix for role | SuperAdmin |

---

## 2. System Configuration (FR-SC-02)

### 2.1 Business Rules

| Rule ID | Business Rule |
|---------|---------------|
| BR-SC-07 | API key dan secret disimpan terenkripsi (AES-256). Tidak dikembalikan plaintext dari API; di-mask (`****`) di UI setelah disimpan. |
| BR-SC-08 | Tombol "Test Connection" tersedia untuk memverifikasi konfigurasi API sebelum disimpan. |
| BR-SC-09 | Perubahan konfigurasi memerlukan konfirmasi sebelum berlaku dan berlaku langsung tanpa restart. |
| BR-SC-10 | Semua perubahan konfigurasi tercatat di audit log. |

### 2.2 Database Schema

```sql
CREATE TABLE system_configs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_key  VARCHAR(100) NOT NULL UNIQUE,
    config_value TEXT NOT NULL,
    value_type  VARCHAR(20) NOT NULL DEFAULT 'string',
    -- string | number | boolean | json
    description TEXT,
    updated_by  UUID REFERENCES admin_users(id),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed defaults:
-- weather.wave_warning_threshold (number, meter)
-- weather.wave_critical_threshold (number, meter)
-- weather.wind_warning_threshold (number, knot)
-- weather.wind_critical_threshold (number, knot)
-- weather.visibility_warning_threshold (number, km)
-- weather.visibility_critical_threshold (number, km)
-- weather.update_interval_minutes (number)
-- data.retention_years (number)
-- ais.message_template_default (string)

CREATE TABLE api_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_name VARCHAR(50) NOT NULL UNIQUE,
    -- INAPORTNET | SIMODEL | WEATHER_API | AIS_PROVIDER | STS_PLATFORM | BEA_CUKAI
    display_name    VARCHAR(100) NOT NULL,
    base_url        TEXT,
    api_key_encrypted TEXT,
    api_secret_encrypted TEXT,
    extra_params    JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_test_at    TIMESTAMPTZ,
    last_test_status VARCHAR(20),
    -- OK | FAILED | UNTESTED
    updated_by      UUID REFERENCES admin_users(id),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_configs (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type   VARCHAR(100) NOT NULL UNIQUE,
    -- WEATHER_WARNING | WEATHER_CRITICAL | VESSEL_ARRIVAL | INCIDENT | PAYMENT_REMINDER | SYSTEM_ALERT
    is_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    recipient_roles TEXT[] NOT NULL DEFAULT '{}',
    -- Array role names yang menerima notifikasi
    updated_by   UUID REFERENCES admin_users(id),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 2.3 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/admin/system-configs | List all system config params | SuperAdmin |
| PUT | /api/admin/system-configs | Bulk update system config params | SuperAdmin |
| GET | /api/admin/api-configs | List all API integrations | SuperAdmin |
| PUT | /api/admin/api-configs/:name | Update API integration config | SuperAdmin |
| POST | /api/admin/api-configs/:name/test | Test connection for integration | SuperAdmin |
| GET | /api/admin/alert-configs | List all alert configs | SuperAdmin |
| PUT | /api/admin/alert-configs | Bulk update alert configs | SuperAdmin |

---

## 3. Equipment Management (FR-SC-03)

### 3.1 Business Rules

| Rule ID | Business Rule |
|---------|---------------|
| BR-SC-11 | Setiap peralatan memiliki status: Operational, Maintenance, Standby, Out of Service. |
| BR-SC-12 | Alert dikirim ke Admin jika peralatan kritis (AIS Station, RADAR, VHF Radio) berstatus non-Operational. |
| BR-SC-13 | History status dan maintenance setiap peralatan tersimpan lengkap (tidak bisa dihapus). |
| BR-SC-14 | Sistem mengirim reminder 7 hari sebelum jadwal maintenance yang dikonfigurasi. |

### 3.2 Database Schema

```sql
CREATE TABLE equipment (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name             VARCHAR(255) NOT NULL,
    equipment_type   VARCHAR(50) NOT NULL,
    -- AIS_STATION | VHF_RADIO | RADAR | CCTV | WEATHER_STATION
    serial_number    VARCHAR(100),
    purchase_date    DATE,
    warranty_expiry  DATE,
    location         VARCHAR(255),
    status           VARCHAR(30) NOT NULL DEFAULT 'OPERATIONAL',
    -- OPERATIONAL | MAINTENANCE | STANDBY | OUT_OF_SERVICE
    next_maintenance_at DATE,
    notes            TEXT,
    is_active        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment_status_history (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id  UUID NOT NULL REFERENCES equipment(id),
    old_status    VARCHAR(30),
    new_status    VARCHAR(30) NOT NULL,
    changed_by    UUID REFERENCES admin_users(id),
    notes         TEXT,
    changed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.3 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/admin/equipment | List all equipment | Admin, SuperAdmin |
| POST | /api/admin/equipment | Create new equipment | Admin, SuperAdmin |
| GET | /api/admin/equipment/:id | Get equipment detail + history | Admin, SuperAdmin |
| PUT | /api/admin/equipment/:id | Update equipment data | Admin, SuperAdmin |
| PUT | /api/admin/equipment/:id/status | Update equipment status | Admin, SuperAdmin |

---

## 4. Audit Log (FR-SC-04)

### 4.1 Business Rules

| Rule ID | Business Rule |
|---------|---------------|
| BR-SC-15 | Audit log bersifat insert-only (immutable). Tidak ada endpoint untuk edit atau delete. |
| BR-SC-16 | Setiap entry mencatat: timestamp, actor (user_id + nama), action, entity_type, entity_id, detail perubahan (sebelum/sesudah dalam JSON). |

### 4.2 Database Schema

```sql
CREATE TABLE audit_logs (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id     UUID REFERENCES admin_users(id),
    actor_name   VARCHAR(255),
    action       VARCHAR(50) NOT NULL,
    -- CREATE | UPDATE | DELETE | LOGIN | LOGOUT | CONFIG_CHANGE | STATUS_CHANGE
    entity_type  VARCHAR(100) NOT NULL,
    entity_id    VARCHAR(255),
    details      JSONB,
    -- { "before": {...}, "after": {...} }
    ip_address   VARCHAR(45),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_actor_id    ON audit_logs(actor_id);
CREATE INDEX idx_audit_logs_entity_type ON audit_logs(entity_type);
CREATE INDEX idx_audit_logs_created_at  ON audit_logs(created_at DESC);
```

### 4.3 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/admin/audit-logs | List audit logs with filters | SuperAdmin |

Query params: `actor_id`, `entity_type`, `action`, `from` (ISO date), `to` (ISO date), `page`, `limit`

---

## 5. STS Data Sync (FR-SC-05, FR-SC-06)

### 5.1 Business Rules

| Rule ID | Business Rule |
|---------|---------------|
| BR-SC-17 | Sync berjalan otomatis secara berkala (interval dikonfigurasi via system_configs). Dapat juga di-trigger manual dari UI. |
| BR-SC-18 | Saat koneksi ke STS terputus, LPS menggunakan data local cache terakhir. Tidak ada downgrade fungsi operasional. |
| BR-SC-19 | Status dan timestamp sync terakhir ditampilkan di halaman konfigurasi. |

### 5.2 Database Schema — Local Cache

```sql
CREATE TABLE sts_vessels_cache (
    id              UUID PRIMARY KEY,
    -- ID dari STS Platform
    name            VARCHAR(255) NOT NULL,
    imo_number      VARCHAR(20),
    mmsi            VARCHAR(20),
    vessel_type     VARCHAR(50),
    flag            VARCHAR(100),
    grt             DECIMAL(10,2),
    loa             DECIMAL(10,2),
    owner_name      VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sts_sync_log (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type  VARCHAR(50) NOT NULL,
    -- VESSEL | STAKEHOLDER | ZONE | ANCHOR_POINT
    status       VARCHAR(20) NOT NULL,
    -- SUCCESS | FAILED | PARTIAL
    records_synced INTEGER DEFAULT 0,
    error_message  TEXT,
    synced_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.3 API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /api/admin/sts-sync/status | Status dan timestamp sync terakhir per entity | SuperAdmin |
| POST | /api/admin/sts-sync/trigger | Trigger manual sync dari STS Platform | SuperAdmin |

---

## 6. Frontend Routes

| Route | Component | Akses |
|-------|-----------|-------|
| /admin/settings/users | UserManagementPage | SuperAdmin |
| /admin/settings/users/new | UserFormPage | SuperAdmin |
| /admin/settings/users/:id | UserDetailPage | SuperAdmin |
| /admin/settings/roles | RolePermissionsPage | SuperAdmin |
| /admin/settings/system | SystemConfigPage | SuperAdmin |
| /admin/settings/api-integrations | ApiIntegrationsPage | SuperAdmin |
| /admin/settings/equipment | EquipmentPage | Admin, SuperAdmin |
| /admin/settings/equipment/new | EquipmentFormPage | Admin, SuperAdmin |
| /admin/settings/equipment/:id | EquipmentDetailPage | Admin, SuperAdmin |
| /admin/settings/audit-logs | AuditLogPage | SuperAdmin |
| /admin/settings/sts-sync | StsSyncPage | SuperAdmin |
