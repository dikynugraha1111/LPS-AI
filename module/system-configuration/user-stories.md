# M12 — System Configuration: User Stories

## US-SC-01: User & Role Management

**As a** SuperAdmin,  
**I want to** manage all admin user accounts and their role assignments,  
**So that** every user can only access the features and data appropriate for their responsibilities.

**Acceptance Criteria:**
- [ ] SuperAdmin can create a new admin user with fields: name, email, username, password, role, and active status
- [ ] SuperAdmin can edit user data and change role assignment
- [ ] SuperAdmin can deactivate a user; deactivated user cannot login and active sessions are immediately invalidated
- [ ] SuperAdmin cannot deactivate their own account
- [ ] Username and email must be unique; system rejects duplicates with an informative error message
- [ ] All user changes are recorded in audit log with actor name and timestamp
- [ ] User list can be searched and filtered by name, role, or status

---

## US-SC-02: Permission Matrix Configuration

**As a** SuperAdmin,  
**I want to** configure the permission matrix for each role per module,  
**So that** access control reflects the operational responsibilities of each role.

**Acceptance Criteria:**
- [ ] SuperAdmin can view and edit the permission matrix for each role (view / create / edit / delete per module)
- [ ] Available modules in the permission matrix: M1, M2, M3, M4, M5, M6, M11, M12
- [ ] Changes to permissions take effect on the user's next login
- [ ] All permission changes are recorded in audit log

---

## US-SC-03: System Parameter & API Configuration

**As a** SuperAdmin,  
**I want to** configure system parameters and external API integrations from the admin dashboard,  
**So that** the system can be adapted to operational needs without changing application code.

**Acceptance Criteria:**
- [ ] SuperAdmin can configure weather thresholds: wave warning/critical (m), wind warning/critical (knot), visibility warning/critical (km)
- [ ] SuperAdmin can configure alert recipients per event type (Weather Warning, Weather Critical, Vessel Arrival, Incident, Payment Reminder, System Alert)
- [ ] SuperAdmin can view the list of all external API integrations with their real-time connection status
- [ ] SuperAdmin can update endpoint URL, API key, and connection parameters for each integration
- [ ] "Test Connection" button is available; test result displayed before saving
- [ ] API key and secret values are masked (`****`) in the UI after being saved; never returned as plaintext
- [ ] All configuration changes require confirmation dialog before taking effect
- [ ] Configuration changes take effect immediately without system restart
- [ ] All configuration changes are recorded in audit log

---

## US-SC-04: Equipment Management

**As an** Admin Kepelabuhan,  
**I want to** manage the inventory and operational status of all LPS equipment,  
**So that** the condition of every device is always monitored and maintenance can be planned.

**Acceptance Criteria:**
- [ ] Admin can create new equipment with fields: name, type (AIS Station / VHF Radio / RADAR / CCTV / Weather Station), serial number, purchase date, warranty expiry, location, and notes
- [ ] Admin can update equipment status: Operational, Maintenance, Standby, Out of Service
- [ ] Status change history for each equipment is stored and cannot be deleted
- [ ] Admin can set a next maintenance date; system sends a reminder 7 days before the scheduled maintenance
- [ ] Alert is sent to Admin when a critical device (AIS Station, RADAR, VHF Radio) becomes non-Operational
- [ ] Equipment list can be filtered by type and status

---

## US-SC-05: Audit Log & STS Sync Monitoring

**As a** SuperAdmin,  
**I want to** view the system activity audit log and monitor STS data synchronization status,  
**So that** I can trace any changes in the system and ensure master data in LPS is always up to date.

**Acceptance Criteria:**
- [ ] SuperAdmin can view audit log with filters: entity type, actor (user), action type, and time range
- [ ] Audit log entries cannot be edited or deleted (immutable)
- [ ] Each audit log entry shows: timestamp, actor name, action, entity type, entity ID, and change details (before/after)
- [ ] SuperAdmin can view the last sync status and timestamp per entity type (vessel, stakeholder, zone, anchor point)
- [ ] SuperAdmin can trigger a manual sync from STS Platform
- [ ] If sync fails, error message is displayed with detail reason
- [ ] LPS continues to operate normally using local cache when STS connection is temporarily unavailable
