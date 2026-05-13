# M12 — System Configuration · UI Design

**Last updated:** 2026-05-13 · **Surface:** B (Internal Operator) · **Status:** ACTIVE

> Wajib dibaca bersama `lps-design-system.md`.

---

## 1. Ringkasan

M12 adalah modul admin internal yang mengelola user/role, system config, API integrations, equipment, audit log, dan STS data sync. Surface B (English, top bar, sidebar nested). Modul ini terletak di section `SETTINGS` dari sidebar Operator.

Module scope lihat `module/system-configuration/README.md`. FR source: FR-SC-01..FR-SC-06.

---

## 2. Page Inventory

Semua di prefix `/admin/settings/`.

| Route | Halaman | Akses |
|---|---|---|
| `/admin/settings` | Settings overview (cards grid → 6 sub-modul) | Admin, SuperAdmin |
| `/admin/settings/users` | User Management list | Admin, SuperAdmin |
| `/admin/settings/users/:id` | User detail / edit | Admin, SuperAdmin |
| `/admin/settings/users/new` | Add new user | Admin, SuperAdmin |
| `/admin/settings/roles` | Role & Permission matrix | SuperAdmin only |
| `/admin/settings/system` | System Config (key-value editable) | SuperAdmin only |
| `/admin/settings/integrations` | API Integrations (STS, Inaportnet, SIMoPEL, dll) | SuperAdmin only |
| `/admin/settings/equipment` | Equipment Management | Operator + Admin |
| `/admin/settings/audit-log` | Audit Log (read-only, filter & search) | Admin, SuperAdmin |
| `/admin/settings/sts-sync` | STS Data Sync status & control | SuperAdmin only |

---

## 3. Settings Layout & Sub-Navigation

Saat user masuk ke `/admin/settings/*`, sidebar **SETTINGS** expanded dan sub-item tampil. Sub-item pattern sama dengan Vessel Monitoring di screenshot operator (UPPERCASE tracked, indented).

```
SETTINGS                ^
  USER MANAGEMENT
  ROLES & PERMISSIONS
  SYSTEM CONFIG
  INTEGRATIONS
  EQUIPMENT
  AUDIT LOG
  STS SYNC
```

Atau alternatif: gunakan **horizontal sub-nav** (tabs underline) di top of main content saat berada di `/admin/settings/*` — lebih fleksibel kalau ingin tetap pakai sidebar untuk grouping besar saja. **Rekomendasi: sub-nav vertikal di sidebar** (konsisten dengan Vessel Monitoring dan Weather pattern).

---

## 4. Halaman: Settings Overview (`/admin/settings`)

**Layout:** Grid cards 3-column. Setiap card = pintu masuk ke sub-modul.

```
Top bar (Surface B) + Sidebar B (SETTINGS expanded)

Page header:
  Settings
  Manage users, roles, system config, integrations, and audit log.

Grid (3 cols):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ [👥]     │  │ [🔐]     │  │ [⚙️]     │
  │ User     │  │ Roles &  │  │ System   │
  │ Manage   │  │ Permis.  │  │ Config   │
  │ 12 users │  │ 9 roles  │  │ 47 keys  │
  └──────────┘  └──────────┘  └──────────┘
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ [🔗]     │  │ [📡]     │  │ [📋]     │
  │ API      │  │ Equip-   │  │ Audit    │
  │ Integr.  │  │ ment     │  │ Log      │
  │ 5 active │  │ 23 items │  │ Last 24h │
  └──────────┘  └──────────┘  └──────────┘
  ┌──────────┐
  │ [🔄]     │
  │ STS Sync │
  │ Last:5m  │
  └──────────┘
```

Per card: white `rounded-2xl border p-6 hover:shadow-md transition`. Icon container `h-12 w-12 rounded-lg bg-slate-100`. Title `text-base font-semibold`. Sub-stat `text-sm text-slate-500 mt-1`. Hover lift effect. Klik → navigate ke sub-modul.

---

## 5. Halaman: User Management (`/admin/settings/users`)

**Layout:** Top bar B + sidebar B. Standard list pattern.

```
Page header:
  User Management
  Manage admin and operator accounts.                  [+ Add User]

Filter bar:
  [Search by name/email...]   [Role: All ▾]   [Status: All ▾]

Table:
  Name         Email                Role          Status   Last Login   Actions
  Andi W.      andi@lps.id          SuperAdmin    Active   2h ago       Edit | Disable
  Budi S.      budi@lps.id          Operator      Active   1d ago       Edit | Disable
  Sari L.      sari@lps.id          Admin         Disabled —            Edit | Enable
```

**Table specifics:**
- Avatar di kolom Name (lingkaran initials atau image).
- Role badge: Info variant.
- Status badge: Success "Active" / Neutral "Disabled".
- Actions kolom: text buttons separated by `|`, atau kebab menu dropdown.

### 5.1 User Detail / Edit

Form sections:
- Account Info: name, email (read-only after create), phone
- Role assignment (select)
- Status toggle
- Password reset action (sends email)
- Activity preview (last 5 audit entries terkait user ini)

### 5.2 Add User

Form: name, email, role select, optional phone. Setelah save, sistem auto-generate password + send welcome email. Toast success "User created. Welcome email sent to {email}."

---

## 6. Halaman: Roles & Permissions (`/admin/settings/roles`)

**Layout:** Matrix table (rows = roles, columns = permission categories).

```
Page header:
  Roles & Permissions
  Configure access matrix for system roles.

Section: Roles
  9 roles (SuperAdmin, Admin, Operator, Finance, Customer Service,
  VTS Operator, BC Officer, Customs Officer, ReadOnly)
  Each row: role name + description + member count + Edit button.

Section: Permission Matrix
  Big scrollable table:
    Row = role
    Columns = permission keys (grouped: Customers, Nominations, EPB, 
              Vessels, Weather, Settings, Audit)
    Cell = checkbox (read / write / admin) atau dot indicator.
```

Hanya SuperAdmin yang bisa edit. Edit mode: cell toggle dengan tri-state (—, ✓ read, ✓ write).

Save action: floating action bar di bottom "You have unsaved changes" + button "Save Changes" primary + "Discard" ghost. Pattern sticky-bottom bar.

---

## 7. Halaman: System Config (`/admin/settings/system`)

**Layout:** Grouped key-value editor.

```
Page header:
  System Config
  Manage application settings.

Tabs (underline):
  [General]  [Notifications]  [Security]  [Branding]

Per tab: list of config keys.
  Each row:
    Key (mono small) + Description (muted) + Current value (input/select/toggle) + Save button
```

**Config types & rendering:**
- String: text input
- Number: number input
- Boolean: shadcn `<Switch>`
- Select (enum): shadcn `<Select>`
- JSON: shadcn `<Textarea>` monospace with syntax highlighting + validate button
- Sensitive (API key): masked input + "Reveal" button (logs to audit) + "Rotate" action

Save per row (inline) atau bulk save di bottom (sticky bar). Audit log entry dibuat setiap save.

---

## 8. Halaman: API Integrations (`/admin/settings/integrations`)

**Layout:** Card list per integration provider.

```
Page header:
  API Integrations
  Manage external system credentials and webhook endpoints.

Cards stacked:
  ┌────────────────────────────────────────────────────┐
  │ STS Platform                            [● Active] │
  │ Base URL: https://api.sts.tbk.co.id                │
  │ API Key: ••••••••••••3a4b                          │
  │ Last sync: 5 min ago    Webhook events: 1,247      │
  │                                                    │
  │ [Edit] [Test Connection] [Rotate Key]              │
  └────────────────────────────────────────────────────┘
  
  (sama untuk Inaportnet, SIMoPEL, Bea Cukai, Weather API)
```

**Card detail:**
- Status indicator dot (green/red/yellow) + label.
- Masked credentials.
- Connection metrics (last sync, requests count).
- Actions: Edit (opens dialog), Test Connection (calls health endpoint, shows toast), Rotate Key (generates new key, archives old, requires confirmation).
- Edit dialog: form dengan field URL, key/secret, optional webhook URL, optional timeout.

---

## 9. Halaman: Equipment Management (`/admin/settings/equipment`)

```
Page header:
  Equipment Management
  Track AIS receivers, radars, and weather stations.    [+ Add Equipment]

Filter bar:
  [Search by name/serial...]  [Type: All ▾]  [Status: All ▾]

Grid cards (3 cols) atau Table view (toggle):
  Per item: name, type icon, serial, location, status indicator, 
            last heartbeat, maintenance due date.
```

Status:
- Active (Success badge)
- Maintenance (Warning badge)
- Offline (Error badge)
- Decommissioned (Neutral badge)

Equipment detail page: history of status changes (timeline), maintenance log, telemetry chart (if applicable).

---

## 10. Halaman: Audit Log (`/admin/settings/audit-log`)

**Layout:** Filter + table view dengan virtual scroll (data besar).

```
Page header:
  Audit Log
  Immutable record of all administrative actions.

Filter bar:
  [Date range picker]
  [Actor: All users ▾]
  [Action: All actions ▾]
  [Resource: All ▾]
  [Search...]
  [Export CSV]

Table:
  Timestamp        Actor         Action               Resource          IP        Details
  2026-05-13 06:21 andi@lps.id  USER.UPDATE          users/12          ...       View
  ...
```

**Per row:**
- Timestamp absolute + relative (hover tooltip).
- Actor: email + role badge mini.
- Action: tagged badge (CREATE green, UPDATE blue, DELETE red, LOGIN gray).
- Resource: link ke resource asal (mis. users/12 → user detail).
- Details: button "View" → dialog dengan diff before/after JSON.

**Immutable enforcement:** no edit/delete actions. Only view + export.

---

## 11. Halaman: STS Data Sync (`/admin/settings/sts-sync`)

**Layout:** Status overview + history.

```
Page header:
  STS Data Sync
  Synchronize master data from STS Platform.

KPI row (4-up):
  [LAST SYNC]    [TOTAL ENTITIES]  [SYNC ERRORS]  [STATUS]
  5 min ago      2,471             0              ● Healthy

Section: Cached Entities
  Table: Entity Type | Count | Last Updated | Source Endpoint | Action
   Vessels           1,247    5 min ago      /api/sts/vessels    Refresh
   Stakeholders      324      5 min ago      ...                 Refresh
   Rate Cards        18       1 hr ago       ...                 Refresh

Section: Sync History (chart + table)
  Bar chart sync per hour (last 24h) — Recharts.
  Table 20 latest sync runs: timestamp, duration, entities updated, status.
```

Action button "Sync Now" primary di header (untuk SuperAdmin). Triggers manual sync, shows loading state + progress feedback.

---

## 12. Component Usage Summary

| Component | Pakai |
|---|---|
| Sidebar B (nested expanded) | Semua settings pages |
| Top bar B | Semua |
| KPI Card | Settings overview cards, STS Sync overview |
| Section Card | Semua halaman |
| Table (data table) | Users, Audit Log, STS Sync history, Equipment table view |
| Tabs underline | System Config, Equipment view toggle |
| Status Badge | Users, Equipment, Integrations, STS Sync |
| Severity Badge | Equipment offline, errors |
| Dialog (shadcn) | Edit integration, Audit detail, Confirm destructive |
| Switch / Select / Input | System Config |
| Chart (Recharts) | STS Sync history |
| Sticky action bar | Roles matrix unsaved changes |

---

## 13. Edge Cases

| Trigger | UI behavior |
|---|---|
| API integration test connection fails | Toast error + tampilkan error detail di card "Connection failed: {message}". |
| Rotate key | Dialog confirmation "Old key will be archived. Are you sure?" + checkbox "I understand this cannot be undone." |
| User disables themselves | Dialog warning "You cannot disable your own account." Prevent action. |
| Audit log empty (new install) | Empty state "No audit entries yet. Actions will appear here once recorded." |
| STS sync running | KPI status dot pulse animate, "Sync in progress..." text. Button "Sync Now" disabled. |

---

## 14. Permission-based UI rendering

Patokan:
- SuperAdmin → semua menu visible & editable.
- Admin → Users (limited: no SuperAdmin assignment), Audit, Equipment, STS Sync read-only.
- Operator → Equipment only (limited write).

Implementation: hide entire nav item jika user role tidak punya read permission. Saat user akses URL langsung yang tidak punya hak: redirect ke `/admin/settings` dengan toast "You don't have permission to access this page."

---

## 15. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- Module scope: `module/system-configuration/`
- Replit handoff: `implementation/replit-handoff/m12-system-configuration.md`
- BRD: `document/brd/m12-system-configuration.md`
